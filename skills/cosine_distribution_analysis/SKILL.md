# Cosine Similarity Distribution Analysis Skill

## Metadata
- **Skill Name**: Cosine Distribution Analysis
- **Version**: 1.0.0
- **Created**: 2026-05-08
- **Language**: SQL (Snowflake View with Dynamic Filtering)
- **Type**: Analytical View

## Objective
Analyzes the distribution of cosine similarity scores by bucketing similarities into predefined ranges and computing aggregate statistics (count, min, max, average) for each bucket. Results are returned as a tabular SQL result set compatible with Snowflake Intelligence charting and visualization tools. Supports dynamic year filtering as an input parameter.

## Input Parameters
- **year_filter** (INT, Optional): The fiscal year to filter results (e.g., 2025, 2024, 2023). If not specified, analyzes all years in the dataset.

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
| fiscal_year | INT | Fiscal year of the analysis period |

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
  YEAR(next_perioddate) AS fiscal_year,
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
  similarity_bucket,
  YEAR(next_perioddate)
ORDER BY
  fiscal_year DESC,
  similarity_bucket ASC;
```

## Usage Examples

### Query All Years
```sql
SELECT * FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION;
```

### Filter by Specific Year (Dynamic Parameter)
```sql
-- Analyze 2025 filing distribution
SELECT * 
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025;
```

```sql
-- Analyze 2024 filing distribution
SELECT * 
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2024;
```

### Compare Multiple Years
```sql
SELECT 
  fiscal_year,
  similarity_bucket,
  filing_count,
  avg_cosine_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2023, 2024, 2025)
ORDER BY fiscal_year DESC, similarity_bucket;
```

### Get Total Filing Count by Year
```sql
SELECT 
  fiscal_year,
  SUM(filing_count) as total_filings,
  COUNT(*) as num_buckets,
  AVG(avg_cosine_similarity) as overall_avg_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
GROUP BY fiscal_year
ORDER BY fiscal_year DESC;
```

### Analyze High Similarity Filings (Year 2025)
```sql
SELECT 
  similarity_bucket,
  filing_count,
  avg_cosine_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025 
  AND similarity_bucket >= '5. 0.98 to 0.99'
ORDER BY similarity_bucket;
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
- **fiscal_year**: Year of the analysis period extracted from `next_perioddate`
- **filing_count**: Number of filings that fall within each similarity bucket for the specified year
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
- `next_perioddate` (DATE/TIMESTAMP): Filing period date for temporal analysis and year filtering
- `lm_cosine_similarity` (FLOAT): Cosine similarity score (typically between 0 and 1)

## Key Features

1. **Dynamic Year Filtering**: Use `WHERE fiscal_year = <year>` to filter results by any year in the dataset
2. **Tabular Result Set**: Returns results as a standard SQL table compatible with Snowflake Intelligence charting
3. **Null Handling**: Automatically filters out rows where `lm_cosine_similarity` is NULL
4. **Ordered Buckets**: Numeric prefixes (1-9) ensure proper sorting in visualizations
5. **Aggregated Statistics**: Provides min, max, average, and count for each bucket
6. **Multi-Year Analysis**: Supports year-over-year comparisons and temporal trends
7. **Temporal Data**: Includes min/max date ranges and fiscal year for comprehensive analysis
8. **Performance**: View-based approach provides efficient querying and caching benefits
9. **Easy Integration**: Direct SQL view that works seamlessly with BI tools and Snowflake Intelligence

## Notes

1. The `fiscal_year` column is automatically extracted from `next_perioddate` for dynamic filtering
2. The numeric prefixes in bucket names ensure correct alphabetical sorting in visualizations
3. All buckets are grouped and aggregated by year - use WHERE clause to filter specific years
4. The view is read-only and automatically updates as source data changes
5. Results are directly compatible with charting tools without additional transformation
6. For year-specific analysis, use `WHERE fiscal_year = <desired_year>` in your query
7. Multiple years can be analyzed simultaneously using `WHERE fiscal_year IN (year1, year2, year3)`
8. Consider adding indexes on `next_perioddate` and `lm_cosine_similarity` in source table for better performance with large datasets
