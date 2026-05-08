# Cosine Similarity Distribution Analysis Skill

## Metadata
- **Skill Name**: Cosine Distribution Analysis
- **Version**: 2.0.0
- **Created**: 2026-05-08
- **Last Updated**: 2026-05-08
- **Language**: SQL (Snowflake View with Dynamic Filtering)
- **Type**: Analytical View for Time Series & Distribution Analysis

## Objective
Analyzes the distribution of cosine similarity scores by bucketing similarities into predefined ranges and computing aggregate statistics (count, min, max, average) for each bucket. Returns tabular SQL result sets optimized for direct visualization in Snowflake Intelligence charting tools without additional transformation.

Supports dynamic year filtering, multi-year comparisons, and trend analysis. Results are immediately ready for bar charts, trend lines, and distribution visualizations.

## Input Parameters
- **fiscal_year** (INT, Optional): The fiscal year to filter results (e.g., 2025, 2024, 2023). If not specified, analyzes all years in the dataset.

## Output Schema
Returns a table with the following columns:

| Column | Type | Description |
|--------|------|-------------|
| similarity_bucket | STRING | Ordered bucket category (e.g., '1. [0.0, 0.90)', '2. [0.90, 0.95)', etc.) |
| filing_count | INT | Number of filings in this bucket (perfect for Y-axis in bar charts) |
| min_cosine_similarity | FLOAT | Minimum cosine similarity value in bucket |
| max_cosine_similarity | FLOAT | Maximum cosine similarity value in bucket |
| avg_cosine_similarity | FLOAT | Average cosine similarity value in bucket (good for trend visualization) |
| min_next_perioddate | DATE | Earliest filing period date in bucket |
| max_next_perioddate | DATE | Latest filing period date in bucket |
| fiscal_year | INT | Fiscal year of the analysis period (use in WHERE clause for filtering) |
| bucket_sort_order | INT | Numeric sort order (1-9) for maintaining proper bucket sequence in charts |

## View Definition

```sql
CREATE OR REPLACE VIEW QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION AS
SELECT
  CASE
    WHEN lm_cosine_similarity < 0.90
    THEN '1. [0.0, 0.90)'
    WHEN lm_cosine_similarity >= 0.90 AND lm_cosine_similarity < 0.95
    THEN '2. [0.90, 0.95)'
    WHEN lm_cosine_similarity >= 0.95 AND lm_cosine_similarity < 0.97
    THEN '3. [0.95, 0.97)'
    WHEN lm_cosine_similarity >= 0.97 AND lm_cosine_similarity < 0.98
    THEN '4. [0.97, 0.98)'
    WHEN lm_cosine_similarity >= 0.98 AND lm_cosine_similarity < 0.99
    THEN '5. [0.98, 0.99)'
    WHEN lm_cosine_similarity >= 0.99 AND lm_cosine_similarity < 0.995
    THEN '6. [0.99, 0.995)'
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999
    THEN '7. [0.995, 0.999)'
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0
    THEN '8. [0.999, 1.0]'
    ELSE '9. Other'
  END AS similarity_bucket,
  CASE
    WHEN lm_cosine_similarity < 0.90 THEN 1
    WHEN lm_cosine_similarity >= 0.90 AND lm_cosine_similarity < 0.95 THEN 2
    WHEN lm_cosine_similarity >= 0.95 AND lm_cosine_similarity < 0.97 THEN 3
    WHEN lm_cosine_similarity >= 0.97 AND lm_cosine_similarity < 0.98 THEN 4
    WHEN lm_cosine_similarity >= 0.98 AND lm_cosine_similarity < 0.99 THEN 5
    WHEN lm_cosine_similarity >= 0.99 AND lm_cosine_similarity < 0.995 THEN 6
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999 THEN 7
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0 THEN 8
    ELSE 9
  END AS bucket_sort_order,
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
  bucket_sort_order,
  YEAR(next_perioddate)
ORDER BY
  fiscal_year DESC,
  bucket_sort_order ASC;
```

## Charting Guide

### Distribution Bar Chart (Recommended for Single Year Analysis)

**Chart Type:** Vertical Bar Chart  
**X-Axis:** `similarity_bucket` (ordered categories)  
**Y-Axis:** `filing_count` (quantitative)  
**Color/Series:** Optional - `fiscal_year` if showing multiple years  
**Sort:** By `bucket_sort_order` (ascending) to maintain bucket sequence

**SQL Query for Charting:**
```sql
SELECT 
  similarity_bucket,
  filing_count,
  bucket_sort_order
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025
ORDER BY bucket_sort_order ASC;
```

**Interpretation:** Shows the distribution of filing counts across similarity ranges. A right-skewed distribution (peak at 0.995-0.999) indicates most companies made minimal changes to risk language year-over-year.

---

### Year-over-Year Comparison Bar Chart

**Chart Type:** Grouped Bar Chart  
**X-Axis:** `similarity_bucket` (ordered categories)  
**Y-Axis:** `filing_count`  
**Series/Groups:** `fiscal_year` (multiple years as separate bars)  
**Sort:** By `bucket_sort_order`

**SQL Query for Charting:**
```sql
SELECT 
  similarity_bucket,
  fiscal_year,
  filing_count,
  bucket_sort_order
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2023, 2024, 2025)
ORDER BY fiscal_year DESC, bucket_sort_order ASC;
```

**Interpretation:** Compare how the distribution of cosine similarities has evolved across years. Helps identify if companies are maintaining consistent language or shifting disclosure practices.

---

### Average Similarity Trend Chart

**Chart Type:** Line Chart with Point Markers  
**X-Axis:** `similarity_bucket` (ordered categories)  
**Y-Axis:** `avg_cosine_similarity` (0.0 to 1.0 scale)  
**Color/Series:** `fiscal_year` (one line per year)  
**Sort:** By `bucket_sort_order`

**SQL Query for Charting:**
```sql
SELECT 
  similarity_bucket,
  fiscal_year,
  avg_cosine_similarity,
  bucket_sort_order
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2023, 2024, 2025)
ORDER BY fiscal_year ASC, bucket_sort_order ASC;
```

**Interpretation:** Shows average similarity values across buckets, revealing whether high-similarity buckets contain filings that are equally similar or have more variance.

---

### Filing Count Summary by Year

**Chart Type:** Pie Chart or Stacked Bar Chart  
**Metric:** Total `filing_count` aggregated by `fiscal_year`

**SQL Query for Charting:**
```sql
SELECT 
  fiscal_year,
  SUM(filing_count) as total_filings,
  ROUND(100.0 * SUM(filing_count) / (SELECT SUM(filing_count) FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION), 2) as pct_of_total
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
GROUP BY fiscal_year
ORDER BY fiscal_year DESC;
```

**Interpretation:** Shows the proportion of all filings coming from each year, helpful for understanding dataset composition.

---

## Usage Examples

### Query All Years (No Filtering)
```sql
SELECT * FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
ORDER BY fiscal_year DESC, bucket_sort_order ASC;
```

### Filter by Specific Year (Dynamic Parameter)
```sql
-- Analyze 2025 filing distribution
SELECT * 
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025
ORDER BY bucket_sort_order ASC;
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
ORDER BY fiscal_year DESC, bucket_sort_order ASC;
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
  AND bucket_sort_order >= 5
ORDER BY bucket_sort_order ASC;
```

### Identify Significant Language Changes (Low Similarity)
```sql
SELECT 
  similarity_bucket,
  filing_count,
  min_cosine_similarity,
  max_cosine_similarity,
  fiscal_year
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE bucket_sort_order <= 3
ORDER BY fiscal_year DESC, bucket_sort_order ASC;
```

## Similarity Buckets Reference

| Bucket | Range | Description | Interpretation |
|--------|-------|-------------|-----------------|
| 1. [0.0, 0.90) | < 0.90 | Very Low | Significant changes in risk language |
| 2. [0.90, 0.95) | [0.90, 0.95) | Low | Substantial language differences |
| 3. [0.95, 0.97) | [0.95, 0.97) | Medium-Low | Moderate language changes |
| 4. [0.97, 0.98) | [0.97, 0.98) | Medium | Some language variation |
| 5. [0.98, 0.99) | [0.98, 0.99) | Medium-High | Minor language changes |
| 6. [0.99, 0.995) | [0.99, 0.995) | High | Minimal language differences |
| 7. [0.995, 0.999) | [0.995, 0.999) | Very High | Nearly identical language |
| 8. [0.999, 1.0] | [0.999, 1.0] | Extremely High | Virtually no language change |
| 9. Other | > 1.0 | Out of Range | Edge cases/data anomalies |

## Key Metrics

- **similarity_bucket**: Categorizes cosine similarity into 9 discrete ranges with numeric prefix (1-9) for natural ordering in charts
- **bucket_sort_order**: Numeric value (1-9) explicitly designed for chart sorting - use this for ORDER BY clauses
- **fiscal_year**: Year of the analysis period extracted from `next_perioddate` - use in WHERE clause for year filtering
- **filing_count**: Number of filings per bucket - primary metric for distribution bar charts
- **min_cosine_similarity**: Minimum value in bucket for range verification
- **max_cosine_similarity**: Maximum value in bucket for range verification
- **avg_cosine_similarity**: Mean similarity in bucket - use for trend analysis across buckets
- **min_next_perioddate**: Earliest filing date - useful for temporal context
- **max_next_perioddate**: Latest filing date - useful for temporal context

## Data Requirements

### Source Table
- Database: `QRSLLM_POC_DB`
- Schema: `HENRY_SCHEMA`
- Table: `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`

### Required Columns
- `next_perioddate` (DATE/TIMESTAMP): Filing period date for temporal analysis and year filtering
- `lm_cosine_similarity` (FLOAT): Cosine similarity score (typically between 0 and 1)

## Key Features

1. **Chart-Ready Results**: Direct SQL output without transformation needed for visualization
2. **Numeric Sort Order**: `bucket_sort_order` column ensures proper charting sequence regardless of string sorting
3. **Dynamic Year Filtering**: Use `WHERE fiscal_year = <year>` to filter results by any year in the dataset
4. **Tabular Result Set**: Returns results as standard SQL table compatible with all Snowflake Intelligence charting
5. **Null Handling**: Automatically filters out rows where `lm_cosine_similarity` is NULL
6. **Multi-Year Analysis**: Supports year-over-year comparisons and temporal trends via `fiscal_year` column
7. **Aggregated Statistics**: Provides min, max, average, and count for each bucket
8. **Temporal Data**: Includes min/max date ranges and fiscal year for comprehensive analysis
9. **Performance**: View-based approach provides efficient querying and automatic caching benefits
10. **Easy Integration**: Works seamlessly with BI tools and Snowflake Intelligence without workarounds

## Quick Start for Charting

**For a simple single-year distribution bar chart:**
```sql
SELECT similarity_bucket, filing_count, bucket_sort_order
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025
ORDER BY bucket_sort_order
```
Then create a **Vertical Bar Chart** with `similarity_bucket` on X-axis and `filing_count` on Y-axis.

**For year-over-year comparison:**
```sql
SELECT fiscal_year, similarity_bucket, filing_count, bucket_sort_order
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2024, 2025)
ORDER BY fiscal_year, bucket_sort_order
```
Then create a **Grouped Bar Chart** with `similarity_bucket` on X-axis, `filing_count` on Y-axis, and `fiscal_year` as the grouping dimension.

## Notes

1. Always include `bucket_sort_order` in queries used for charting - even if not displayed, it ensures correct ordering
2. The numeric prefixes in bucket names (1-9) provide secondary ordering for string-based sorting fallbacks
3. All buckets are grouped and aggregated by year - use WHERE clause to filter specific years
4. The view is read-only and automatically updates as source data changes
5. Results are directly compatible with charting tools without additional transformation
6. For year-specific analysis, use `WHERE fiscal_year = <desired_year>` in your query
7. Multiple years can be analyzed simultaneously using `WHERE fiscal_year IN (year1, year2, year3)`
8. Consider adding indexes on `next_perioddate` and `lm_cosine_similarity` in source table for improved query performance with large datasets
