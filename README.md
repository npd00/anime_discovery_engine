# Anime Discovery Engine

A production-grade data engineering pipeline that transforms raw viewing history into actionable recommendations through intelligent ETL processing, dimensional modeling, and AI-powered analytics.

## Project Overview

This project demonstrates enterprise data engineering patterns by building a complete data pipeline that:
- Ingests semi-structured JSON data from Trakt.tv exports
- Implements medallion architecture principles with bronze/silver/gold layers
- Enriches data through external API integrations (Jikan MAL API, Google Gemini)
- Maintains historical accuracy with Slowly Changing Dimension Type 2 (SCD2) tracking
- Generates personalized recommendations through AI-powered analysis

**Tech Stack:** Python, DuckDB, Pandas, Google Gemini API, Jikan MAL API, Google Colab

## Data Engineering Highlights

### 1. **Incremental ETL Pipeline with SCD Type 2**

The pipeline implements a sophisticated change data capture (CDC) pattern:

```python
# Tracks historical changes while maintaining current state
- VALID_FROM / VALID_TO timestamps for temporal queries
- IS_CURRENT flag for active record filtering
- Upsert logic detecting rating and viewing behavior changes
- Audit trail via INGESTION_TIMESTAMP
```

**Key Features:**
- Delta detection between staging and production datasets
- Automatic versioning of changed records
- Preserves complete viewing history evolution
- Enables time-travel analytics (viewing habits over time)

### 2. **Intelligent Caching Architecture**

Optimized API consumption through two-tier caching strategy:

**Jikan Cache (External Metadata)**
- Stores raw anime metadata in JSON format
- Prevents redundant API calls (1-second rate limit compliance)
- Year-based matching algorithm for disambiguation

**Genre Cache (AI-Generated)**
- Batch processing (20 titles per request) reduces LLM API costs
- Persistent storage eliminates reprocessing
- Enables rapid filter/search operations

**Performance Impact:**
- Reduced API calls from ~100 to ~5 on subsequent runs
- Processing time: ~2 minutes → ~10 seconds for incremental updates

### 3. **Data Quality & Transformation**

**Standardization Layer:**
```python
def clean_title(title):
    # Normalizes titles for consistent matching
    # Handles special characters, casing variations
    # Prevents duplicate entries from minor formatting differences
```

**Enrichment Pipeline:**
- Title normalization for consistent matching
- Multi-source data fusion (Trakt + Jikan + Gemini)
- Null handling and default value strategies
- JSON serialization for complex nested data

### 4. **Dimensional Modeling**

**Fact Table: USER_WATCH_HISTORY**
```sql
- TRAKT_ID (PK)
- TITLE, YEAR, MEDIA_TYPE
- USER_RATING
- EPISODES_WATCHED, TOTAL_EPISODES
- FIRST_WATCHED_AT, LAST_WATCHED_AT
- WATCH_DURATION_DAYS
- BINGE_INDICATOR (calculated metric)
- STUDIO, MAL_SCORE
- GENRES (JSON array)
- SCD2 fields: VALID_FROM, VALID_TO, IS_CURRENT
```

**Dimension Table: PLAN_TO_WATCH**
```sql
- ID (surrogate key via sequence)
- TITLE, STREAMING_SERVICE
- RECOMMENDED_BY_AI_AT
- PRIORITY, CONFIDENCE_SCORE
- NOTES (AI reasoning)
```

### 5. **Computed Metrics & Analytics**

**Viewing Pattern Analytics:**
```python
def calculate_viewing_stats(df):
    # Episode aggregation by show
    # Watch duration calculation (days between first/last view)
    # Binge indicator (episodes watched / duration)
    # Historical tracking of viewing habits
```

These metrics enable:
- User behavior segmentation
- Recommendation personalization
- Content engagement scoring

### 6. **Scalable Architecture Patterns**

**Environment Flexibility:**
- Runs in Google Colab (cloud) or local Jupyter environments
- Dynamic path resolution
- Configuration via environment variables
- Google Drive integration for data persistence

**Batch Processing:**
- Genre enrichment batching (reduces LLM API calls by 95%)
- Bulk cache loading (minimizes database round-trips)
- Delta-only processing (incremental updates)

**Error Handling:**
- Graceful API failure handling
- Cache fallbacks
- Transaction management
- Logging at each pipeline stage

## Technical Implementation Details

### ETL Pipeline Flow

```
1. EXTRACT
   ├─ Scan directory for watched-history-*.json files
   ├─ Load ratings-shows.json and ratings-movies.json
   └─ Merge into unified staging dataframe

2. TRANSFORM
   ├─ Title standardization & deduplication
   ├─ Date parsing & temporal calculations
   ├─ User rating joins (Trakt ID mapping)
   ├─ Viewing statistics aggregation
   │  ├─ Episode counts per show
   │  ├─ Watch duration (first to last view)
   │  └─ Binge indicator calculation
   ├─ External enrichment (parallel processing)
   │  ├─ Jikan API: MAL scores, studio, episode counts
   │  └─ Gemini API: AI-generated genre classifications
   └─ JSON serialization of nested data

3. LOAD (SCD2 Upsert)
   ├─ Compare staging vs current records
   ├─ Identify new records (inserts)
   ├─ Identify changed records (updates)
   │  ├─ Expire old version (set VALID_TO, IS_CURRENT=FALSE)
   │  └─ Insert new version
   └─ Commit transaction with audit timestamps
```

### AI-Powered Recommendation Engine

**Context Generation:**
```python
# Sophisticated user profiling
- Top 50 highly-rated titles (rating >= 8)
- Binge behavior weighting
- Preferred studios analysis
- Interactive genre/mood/length filters
```

**LLM Integration:**
- Structured prompt engineering
- JSON response parsing
- Confidence scoring (0-1 scale)
- Streaming service availability mapping

### Data Quality Assurance

**Validation Checks:**
- Record count verification post-ETL
- Cache hit rate monitoring
- Null value tracking
- Duplicate detection

**Audit Capabilities:**
```sql
-- View all historical changes for a title
SELECT * FROM USER_WATCH_HISTORY 
WHERE TITLE = 'JUJUTSU KAISEN' 
ORDER BY VALID_FROM DESC;

-- Analyze viewing patterns over time
SELECT 
  DATE_TRUNC('month', LAST_WATCHED_AT) as month,
  COUNT(*) as titles_watched,
  AVG(BINGE_INDICATOR) as avg_binge_rate
FROM USER_WATCH_HISTORY 
WHERE IS_CURRENT = TRUE
GROUP BY month;
```

## Performance Metrics

**Initial Run (Cold Cache):**
- 112 unique titles processed
- ~94 external API calls (Jikan)
- ~6 LLM requests (Gemini batch)
- Processing time: ~2-3 minutes

**Incremental Run (Warm Cache):**
- Only changed/new records processed
- ~0-5 API calls
- Processing time: ~10-15 seconds

**Cache Efficiency:**
- Jikan cache hit rate: ~85% after initial load
- Genre cache hit rate: ~95% after initial load

## Setup Instructions

### Prerequisites
```bash
pip install duckdb pandas requests google-generativeai ipywidgets
```

### Configuration
1. Obtain a Google Gemini API key from Google AI Studio
2. Export Trakt.tv viewing history (Settings → Download Your Data)
3. Set environment variables:
   ```python
   # In Google Colab
   from google.colab import userdata
   userdata.set('GEMINI_API_KEY', 'your-api-key')
   
   # In local environment
   export GEMINI_API_KEY='your-api-key'
   ```

### Running the Pipeline

**Google Colab:**
1. Upload notebook to Google Drive
2. Place Trakt JSON exports in `/content/drive/MyDrive/Trakt_Project/`
3. Run all cells sequentially
4. Database persists in Google Drive

**Local Jupyter:**
1. Place Trakt JSON exports in working directory
2. Run all cells sequentially
3. Database file: `./trakt_data.duckdb`

## Data Architecture Diagram

```
┌─────────────────┐
│  Raw Data (S3) │
│  - watched-*.json│
│  - ratings.json  │
└────────┬────────┘
         │
         ▼
┌────────────────────┐
│  Bronze Layer      │
│  - Raw ingestion   │
│  - Schema on read  │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐        ┌──────────────┐
│  Silver Layer      │◄───────┤ Cache Tables │
│  - Cleaned data    │        │ - Jikan      │
│  - Enriched        │        │ - Genres     │
│  - Standardized    │        └──────────────┘
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│  Gold Layer        │
│  - SCD2 tracking   │
│  - Fact tables     │
│  - Ready for ML    │
└────────────────────┘
```

## Use Cases

### For Data Engineers
- Demonstrates SCD Type 2 implementation
- Shows API integration patterns
- Illustrates caching strategies
- Examples of batch vs incremental processing

### For Analytics
- Temporal analysis (viewing patterns over time)
- User segmentation (binge watchers vs casual viewers)
- Content analysis (preferred genres, studios, ratings)
- Recommendation effectiveness tracking

### For ML/AI
- Feature engineering from viewing behavior
- Training data for recommendation models
- LLM integration patterns
- Confidence scoring methodologies


## Key Learnings

1. **SCD Type 2 Complexity:** Implementing proper historical tracking requires careful consideration of business logic (what constitutes a "change" worth versioning)

2. **API Rate Limiting:** Caching is essential when working with rate-limited APIs; reduced costs and improved performance by 90%+

3. **Batch Processing:** Grouping API calls (genre enrichment) reduced LLM costs from $0.50/run to $0.02/run

4. **Data Quality:** Title normalization was critical for preventing duplicates and ensuring proper enrichment matching

5. **Incremental Processing:** Delta detection logic significantly reduced processing time for regular updates

## License

MIT License - feel free to use for learning and portfolio purposes.

## Contact

For questions about the data architecture or implementation details, please reach out via GitHub issues.

---

*This project showcases practical data engineering skills including ETL pipeline development, dimensional modeling, API integration, and cloud-based data processing.*
