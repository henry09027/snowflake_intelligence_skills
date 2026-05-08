# Cosine Similarity Distribution Analysis Skill

## Metadata
- **Skill Name**: Cosine Distribution Analysis
- **Version**: 1.0.0
- **Created**: 2026-05-08
- **Language**: Python (Snowflake Snowpark)
- **Runtime**: Python 3.9

## Objective
Analyzes the distribution of cosine similarity scores for filings in a given year by bucketing similarities into predefined ranges and computing aggregate statistics (count, min, max, average) for each bucket.

## Input Parameters
- **year_input** (INT): The year for which to analyze cosine similarity distribution

## Output
Returns a VARIANT (array of objects) containing distribution buckets with the following structure:

```json
[
  {
    "similarity_bucket": "string (e.g., '[0.90, 0.95)')",
    "filing_count": "number of filings in this bucket",
    "min_similarity": "minimum cosine similarity in bucket",
    "max_similarity": "maximum cosine similarity in bucket",
    "avg_similarity": "average cosine similarity in bucket"
  }
]
```

## Procedure Definition

```sql
CREATE OR REPLACE PROCEDURE QRSLLM_POC_DB.HENRY_SCHEMA.GET_COSINE_SIMILARITY_DISTRIBUTION(YEAR_INPUT INT)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'main'
EXECUTE AS OWNER
AS $$
def main(session, year_input: int):
    from snowflake.snowpark.functions import col, when, count, min, max, avg, year, lit
    base_table = "QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned"
    df = (
        session.table(base_table)
        .filter(
            (year(col("next_perioddate")) == year_input) &
            (col("lm_cosine_similarity").is_not_null())
        )
        .with_column(
            "similarity_bucket",
            when(col("lm_cosine_similarity") < 0.90,  lit("[0.0, 0.90)"))
            .when(col("lm_cosine_similarity") < 0.95,  lit("[0.90, 0.95)"))
            .when(col("lm_cosine_similarity") < 0.97,  lit("[0.95, 0.97)"))
            .when(col("lm_cosine_similarity") < 0.98,  lit("[0.97, 0.98)"))
            .when(col("lm_cosine_similarity") < 0.99,  lit("[0.98, 0.99)"))
            .when(col("lm_cosine_similarity") < 0.995, lit("[0.99, 0.995)"))
            .when(col("lm_cosine_similarity") < 0.999, lit("[0.995, 0.999)"))
            .when(col("lm_cosine_similarity") <= 1.01, lit("[0.999, 1.0]"))
            .otherwise(lit("Other"))
        )
        .group_by("similarity_bucket")
        .agg(
            count("*").alias("filing_count"),
            min(col("lm_cosine_similarity")).alias("min_similarity"),
            max(col("lm_cosine_similarity")).alias("max_similarity"),
            avg(col("lm_cosine_similarity")).alias("avg_similarity"),
        )
        .sort("similarity_bucket")
    )
    rows = df.collect()
    result = [
        {
            "similarity_bucket": row["SIMILARITY_BUCKET"],
            "filing_count":      row["FILING_COUNT"],
            "min_similarity":    row["MIN_SIMILARITY"],
            "max_similarity":    row["MAX_SIMILARITY"],
            "avg_similarity":    row["AVG_SIMILARITY"],
        }
        for row in rows
    ]
    return result
$$;
```

## Usage Examples

### Basic Call
```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.GET_COSINE_SIMILARITY_DISTRIBUTION(2024);
```

### Flattening Results into a Table
```sql
SELECT 
    f.value:similarity_bucket::STRING as similarity_bucket,
    f.value:filing_count::INT as filing_count,
    f.value:min_similarity::FLOAT as min_similarity,
    f.value:max_similarity::FLOAT as max_similarity,
    f.value:avg_similarity::FLOAT as avg_similarity
FROM TABLE(FLATTEN(INPUT => QRSLLM_POC_DB.HENRY_SCHEMA.GET_COSINE_SIMILARITY_DISTRIBUTION(2024))) f;
```

## Similarity Buckets Reference

| Bucket | Range | Description |
|--------|-------|-------------|
| [0.0, 0.90) | Very Low | Similarity scores below 90% |
| [0.90, 0.95) | Low | Similarity scores 90-95% |
| [0.95, 0.97) | Medium-Low | Similarity scores 95-97% |
| [0.97, 0.98) | Medium | Similarity scores 97-98% |
| [0.98, 0.99) | Medium-High | Similarity scores 98-99% |
| [0.99, 0.995) | High | Similarity scores 99-99.5% |
| [0.995, 0.999) | Very High | Similarity scores 99.5-99.9% |
| [0.999, 1.0] | Extremely High | Similarity scores 99.9-100% |

## Key Metrics

- **similarity_bucket**: Categorizes cosine similarity into discrete ranges for distribution analysis
- **filing_count**: Number of filings that fall within each similarity bucket
- **min_similarity**: Minimum cosine similarity value observed in the bucket
- **max_similarity**: Maximum cosine similarity value observed in the bucket
- **avg_similarity**: Average (mean) cosine similarity value in the bucket

## Data Requirements

### Source Table
- Database: `QRSLLM_POC_DB`
- Schema: `HENRY_SCHEMA`
- Table: `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`

### Required Columns
- `next_perioddate` (DATE/TIMESTAMP): Filing period date used for year filtering
- `lm_cosine_similarity` (FLOAT): Cosine similarity score (typically between 0 and 1)

## Notes

1. **Null Handling**: The procedure filters out rows where `lm_cosine_similarity` is NULL
2. **Year Filtering**: Only filings from the specified year are included (based on `next_perioddate`)
3. **Bucket Boundaries**: The last bucket includes values up to 1.01 to accommodate potential floating-point precision issues
4. **Sorting**: Results are sorted by similarity bucket for logical progression through ranges
5. **Performance**: Consider adding indexes on `next_perioddate` and `lm_cosine_similarity` for large datasets
6. **Ownership**: Procedure executes as OWNER for data access privileges
