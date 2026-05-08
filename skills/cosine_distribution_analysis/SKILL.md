# Cosine Similarity Distribution Analysis Skill

## Metadata
- **Skill Name**: Cosine Distribution Analysis
- **Version**: 2.1.0
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
| similarity_bucket | STRING | Ordered bucket category (e.g., '[0.0, 0.90)', '[0.90, 0.95)', etc.) |
| filing_count | INT | Number of filings in this bucket (perfect for Y-axis in bar charts) |
| min_similarity | FLOAT | Minimum cosine similarity value in bucket |
| max_similarity | FLOAT | Maximum cosine similarity value in bucket |
| avg_similarity | FLOAT | Average cosine similarity value in bucket (good for trend visualization) |
| min_next_perioddate | DATE | Earliest filing period date in bucket |
| max_next_perioddate | DATE | Latest filing period date in bucket |
| fiscal_year | INT | Fiscal year of the analysis period (use in WHERE clause for filtering) |

## View Definition

```sql
CREATE OR REPLACE VIEW QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION AS
SELECT
  CASE
    WHEN lm_cosine_similarity < 0.90
    THEN '[0.0, 0.90)'
    WHEN lm_cosine_similarity >= 0.90 AND lm_cosine_similarity < 0.95
    THEN '[0.90, 0.95)'
    WHEN lm_cosine_similarity >= 0.95 AND lm_cosine_similarity < 0.97
    THEN '[0.95, 0.97)'
    WHEN lm_cosine_similarity >= 0.97 AND lm_cosine_similarity < 0.98
    THEN '[0.97, 0.98)'
    WHEN lm_cosine_similarity >= 0.98 AND lm_cosine_similarity < 0.99
    THEN '[0.98, 0.99)'
    WHEN lm_cosine_similarity >= 0.99 AND lm_cosine_similarity < 0.995
    THEN '[0.99, 0.995)'
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999
    THEN '[0.995, 0.999)'
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0
    THEN '[0.999, 1.0]'
    ELSE 'Other'
  END AS similarity_bucket,
  YEAR(next_perioddate) AS fiscal_year,
  COUNT(*) AS filing_count,
  MIN(lm_cosine_similarity) AS min_similarity,
  MAX(lm_cosine_similarity) AS max_similarity,
  AVG(lm_cosine_similarity) AS avg_similarity,
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
  CASE
    WHEN similarity_bucket = '[0.0, 0.90)' THEN 1
    WHEN similarity_bucket = '[0.90, 0.95)' THEN 2
    WHEN similarity_bucket = '[0.95, 0.97)' THEN 3
    WHEN similarity_bucket = '[0.97, 0.98)' THEN 4
    WHEN similarity_bucket = '[0.98, 0.99)' THEN 5
    WHEN similarity_bucket = '[0.99, 0.995)' THEN 6
    WHEN similarity_bucket = '[0.995, 0.999)' THEN 7
    WHEN similarity_bucket = '[0.999, 1.0]' THEN 8
    ELSE 9
  END ASC;
```

## Charting Guide

### Distribution Bar Chart (Recommended for Single Year Analysis)

**Chart Type:** Vertical Bar Chart  
**X-Axis:** `similarity_bucket` (ordinal - ordered categories)  
**Y-Axis:** `filing_count` (quantitative)  
**Encoding Type:** `ordinal` for similarity_bucket, `quantitative` for filing_count  

**Direct SQL Query for Charting (Copy-Paste Ready):**
```sql
SELECT 
  similarity_bucket,
  filing_count,
  avg_similarity,
  min_similarity,
  max_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025
ORDER BY 
  CASE
    WHEN similarity_bucket = '[0.0, 0.90)' THEN 1
    WHEN similarity_bucket = '[0.90, 0.95)' THEN 2
    WHEN similarity_bucket = '[0.95, 0.97)' THEN 3
    WHEN similarity_bucket = '[0.97, 0.98)' THEN 4
    WHEN similarity_bucket = '[0.98, 0.99)' THEN 5
    WHEN similarity_bucket = '[0.99, 0.995)' THEN 6
    WHEN similarity_bucket = '[0.995, 0.999)' THEN 7
    WHEN similarity_bucket = '[0.999, 1.0]' THEN 8
    ELSE 9
  END ASC;
```

**Vega-Lite Configuration:**
```json
{
  "encoding": {
    "x": {
      "field": "similarity_bucket",
      "title": "Cosine Similarity Bucket",
      "type": "ordinal",
      "axis": {"labelAngle": -45}
    },
    "y": {
      "field": "filing_count",
      "title": "Number of Filings",
      "type": "quantitative"
    },
    "tooltip": [
      {"field": "similarity_bucket", "type": "ordinal"},
      {"field": "filing_count", "type": "quantitative"},
      {"field": "avg_similarity", "type": "quantitative", "format": ".4f"}
    ]
  },
  "mark": "bar",
  "title": "LM Cosine Similarity Distribution - FY 2025"
}
```

**Interpretation:** Shows the distribution of filing counts across similarity ranges. A right-skewed distribution (peak at 0.995-0.999) indicates most companies made minimal changes to risk language year-over-year.

---

### Year-over-Year Grouped Comparison

**Chart Type:** Grouped/Clustered Bar Chart  
**X-Axis:** `similarity_bucket` (ordinal)  
**Y-Axis:** `filing_count` (quantitative)  
**Color/Series:** `fiscal_year` (nominal - separate bars per year)

**Direct SQL Query for Charting:**
```sql
SELECT 
  similarity_bucket,
  fiscal_year,
  filing_count,
  avg_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2023, 2024, 2025)
ORDER BY fiscal_year DESC,
  CASE
    WHEN similarity_bucket = '[0.0, 0.90)' THEN 1
    WHEN similarity_bucket = '[0.90, 0.95)' THEN 2
    WHEN similarity_bucket = '[0.95, 0.97)' THEN 3
    WHEN similarity_bucket = '[0.97, 0.98)' THEN 4
    WHEN similarity_bucket = '[0.98, 0.99)' THEN 5
    WHEN similarity_bucket = '[0.99, 0.995)' THEN 6
    WHEN similarity_bucket = '[0.995, 0.999)' THEN 7
    WHEN similarity_bucket = '[0.999, 1.0]' THEN 8
    ELSE 9
  END ASC;
```

**Vega-Lite Configuration:**
```json
{
  "encoding": {
    "x": {
      "field": "similarity_bucket",
      "title": "Cosine Similarity Bucket",
      "type": "ordinal",
      "axis": {"labelAngle": -45}
    },
    "y": {
      "field": "filing_count",
      "title": "Number of Filings",
      "type": "quantitative"
    },
    "color": {
      "field": "fiscal_year",
      "type": "nominal",
      "title": "Fiscal Year"
    },
    "tooltip": [
      {"field": "similarity_bucket", "type": "ordinal"},
      {"field": "fiscal_year", "type": "nominal"},
      {"field": "filing_count", "type": "quantitative"}
    ]
  },
  "mark": "bar",
  "title": "YoY Cosine Similarity Comparison"
}
```

**Interpretation:** Compare how the distribution of cosine similarities has evolved across years. Helps identify if companies are maintaining consistent language or shifting disclosure practices.

---

### Average Similarity Trend Line

**Chart Type:** Line Chart with Point Markers  
**X-Axis:** `similarity_bucket` (ordinal)  
**Y-Axis:** `avg_similarity` (quantitative, range 0.0-1.0)  
**Color/Series:** `fiscal_year` (nominal - one line per year)

**Direct SQL Query for Charting:**
```sql
SELECT 
  similarity_bucket,
  fiscal_year,
  avg_similarity,
  filing_count
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2023, 2024, 2025)
ORDER BY fiscal_year ASC,
  CASE
    WHEN similarity_bucket = '[0.0, 0.90)' THEN 1
    WHEN similarity_bucket = '[0.90, 0.95)' THEN 2
    WHEN similarity_bucket = '[0.95, 0.97)' THEN 3
    WHEN similarity_bucket = '[0.97, 0.98)' THEN 4
    WHEN similarity_bucket = '[0.98, 0.99)' THEN 5
    WHEN similarity_bucket = '[0.99, 0.995)' THEN 6
    WHEN similarity_bucket = '[0.995, 0.999)' THEN 7
    WHEN similarity_bucket = '[0.999, 1.0]' THEN 8
    ELSE 9
  END ASC;
```

**Vega-Lite Configuration:**
```json
{
  "encoding": {
    "x": {
      "field": "similarity_bucket",
      "title": "Cosine Similarity Bucket",
      "type": "ordinal",
      "axis": {"labelAngle": -45}
    },
    "y": {
      "field": "avg_similarity",
      "title": "Average Cosine Similarity",
      "type": "quantitative",
      "scale": {"domain": [0.7, 1.0]}
    },
    "color": {
      "field": "fiscal_year",
      "type": "nominal",
      "title": "Fiscal Year"
    },
    "detail": {"field": "fiscal_year"},
    "tooltip": [
      {"field": "similarity_bucket", "type": "ordinal"},
      {"field": "fiscal_year", "type": "nominal"},
      {"field": "avg_similarity", "type": "quantitative", "format": ".4f"},
      {"field": "filing_count", "type": "quantitative"}
    ]
  },
  "mark": {"type": "line", "point": true, "interpolate": "monotone"},
  "title": "Average Similarity Trend by Bucket"
}
```

**Interpretation:** Shows average similarity values across buckets, revealing whether high-similarity buckets contain filings with consistent similarity or more variance.

---

### Filing Count Summary by Year (Pie Chart)

**Chart Type:** Pie Chart  
**Value:** `total_filings` (aggregated)  
**Color:** `fiscal_year` (nominal)

**Direct SQL Query for Charting:**
```sql
SELECT 
  fiscal_year,
  SUM(filing_count) as total_filings,
  ROUND(100.0 * SUM(filing_count) / 
    (SELECT SUM(filing_count) FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION), 2) as pct_of_total
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
GROUP BY fiscal_year
ORDER BY fiscal_year DESC;
```

**Vega-Lite Configuration:**
```json
{
  "encoding": {
    "theta": {
      "field": "total_filings",
      "type": "quantitative"
    },
    "color": {
      "field": "fiscal_year",
      "type": "nominal",
      "title": "Fiscal Year"
    },
    "tooltip": [
      {"field": "fiscal_year", "type": "nominal"},
      {"field": "total_filings", "type": "quantitative"},
      {"field": "pct_of_total", "type": "quantitative", "format": ".1f"}
    ]
  },
  "mark": "arc",
  "title": "Filing Count Distribution by Year"
}
```

**Interpretation:** Shows the proportion of all filings coming from each year, helpful for understanding dataset composition and relative analysis sizes.

---

## Usage Examples

### Query All Years (No Filtering)
```sql
SELECT * FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
ORDER BY fiscal_year DESC;
```

### Filter by Specific Year (Dynamic Parameter)
```sql
-- Analyze 2025 filing distribution
SELECT * 
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025;
```

### Compare Multiple Years
```sql
SELECT 
  fiscal_year,
  similarity_bucket,
  filing_count,
  avg_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year IN (2023, 2024, 2025);
```

### Get Total Filing Count by Year
```sql
SELECT 
  fiscal_year,
  SUM(filing_count) as total_filings,
  COUNT(*) as num_buckets,
  AVG(avg_similarity) as overall_avg_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
GROUP BY fiscal_year
ORDER BY fiscal_year DESC;
```

### Analyze High Similarity Filings (Year 2025)
```sql
SELECT 
  similarity_bucket,
  filing_count,
  avg_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025 
  AND similarity_bucket IN ('[0.98, 0.99)', '[0.99, 0.995)', '[0.995, 0.999)', '[0.999, 1.0]');
```

### Identify Significant Language Changes (Low Similarity)
```sql
SELECT 
  similarity_bucket,
  filing_count,
  min_similarity,
  max_similarity,
  fiscal_year
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE similarity_bucket IN ('[0.0, 0.90)', '[0.90, 0.95)', '[0.95, 0.97)')
ORDER BY fiscal_year DESC;
```

## Similarity Buckets Reference

| Bucket | Range | Description | Interpretation |
|--------|-------|-------------|-----------------|
| [0.0, 0.90) | < 0.90 | Very Low | Significant changes in risk language |
| [0.90, 0.95) | [0.90, 0.95) | Low | Substantial language differences |
| [0.95, 0.97) | [0.95, 0.97) | Medium-Low | Moderate language changes |
| [0.97, 0.98) | [0.97, 0.98) | Medium | Some language variation |
| [0.98, 0.99) | [0.98, 0.99) | Medium-High | Minor language changes |
| [0.99, 0.995) | [0.99, 0.995) | High | Minimal language differences |
| [0.995, 0.999) | [0.995, 0.999) | Very High | Nearly identical language |
| [0.999, 1.0] | [0.999, 1.0] | Extremely High | Virtually no language change |

## Key Metrics

- **similarity_bucket**: Ordered bucket category showing the cosine similarity range (e.g., '[0.0, 0.90)', '[0.90, 0.95)', etc.)
- **fiscal_year**: Year extracted from `next_perioddate` - use in WHERE clause for year filtering
- **filing_count**: Number of filings per bucket - primary metric for distribution bar charts
- **min_similarity**: Minimum cosine similarity value in bucket for range verification
- **max_similarity**: Maximum cosine similarity value in bucket for range verification
- **avg_similarity**: Mean similarity in bucket - use for trend analysis across buckets and in line charts
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

1. **Chart-Ready Column Names**: Output columns match charting tool expectations exactly (similarity_bucket, filing_count, avg_similarity, etc.)
2. **Direct SQL to Vega-Lite**: Queries output flat tabular data compatible with standard JSON charting formats
3. **Proper Ordering**: Results ordered by similarity bucket naturally without requiring additional sorting steps
4. **Dynamic Year Filtering**: Use `WHERE fiscal_year = <year>` to filter results by any year in the dataset
5. **Multiple Visualization Types**: Pre-configured SQL queries and Vega-Lite configs for 4+ chart types
6. **Null Handling**: Automatically filters out rows where `lm_cosine_similarity` is NULL
7. **Multi-Year Analysis**: Supports year-over-year comparisons and temporal trends
8. **Aggregated Statistics**: Provides min, max, average, and count for statistical analysis
9. **Performance**: View-based approach provides efficient querying and automatic caching
10. **Seamless Integration**: Works with Snowflake Intelligence and any Vega-Lite compatible charting system

## Quick Start for Charting

**Minimal Query for Bar Chart:**
```sql
SELECT similarity_bucket, filing_count, avg_similarity, min_similarity, max_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE fiscal_year = 2025
ORDER BY 
  CASE
    WHEN similarity_bucket = '[0.0, 0.90)' THEN 1
    WHEN similarity_bucket = '[0.90, 0.95)' THEN 2
    WHEN similarity_bucket = '[0.95, 0.97)' THEN 3
    WHEN similarity_bucket = '[0.97, 0.98)' THEN 4
    WHEN similarity_bucket = '[0.98, 0.99)' THEN 5
    WHEN similarity_bucket = '[0.99, 0.995)' THEN 6
    WHEN similarity_bucket = '[0.995, 0.999)' THEN 7
    WHEN similarity_bucket = '[0.999, 1.0]' THEN 8
    ELSE 9
  END;
```

Then create a **Vertical Bar Chart** in your charting tool:
- X-axis: `similarity_bucket` (type: ordinal)
- Y-axis: `filing_count` (type: quantitative)
- Tooltips: Include `avg_similarity`, `min_similarity`, `max_similarity`

## Notes

1. Column names match exactly what charting functions expect (no alias transformation needed)
2. Similarity buckets are ordered naturally and will sort correctly in ordinal axes
3. All aggregations are by year - use WHERE clause to filter specific years
4. The view automatically updates as source data changes
5. Include tooltips with `avg_similarity`, `min_similarity`, `max_similarity` for richer insights
6. For year-specific analysis, use `WHERE fiscal_year = <desired_year>`
7. Multiple years can be analyzed simultaneously using `WHERE fiscal_year IN (year1, year2, year3)`
8. The CASE statement for sorting ensures correct bucket ordering even when results are JSON-transformed
9. Consider adding indexes on `next_perioddate` and `lm_cosine_similarity` in source table for improved performance with large datasets
