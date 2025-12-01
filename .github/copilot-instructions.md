# AI Coding Agent Instructions

## Project Overview

This is an educational data orchestration and testing project demonstrating:
- **Dagster** orchestration for data pipelines (3 jobs: pandas, pipeline_one, dbt_pipeline)
- **dbt** transformation with BigQuery backend
- **Great Expectations** for data quality validation
- **dbt_utils** and **dbt_expectations** packages for comprehensive testing

The project teaches orchestration workflows and data quality patterns for unit 2.7 in a data engineering curriculum.

## Architecture & Data Flow

**Three main orchestration patterns** live in `extra/`:

1. **Dagster + dbt** (`dagster_orchestration_dbt/`): Demonstrates asset-based orchestration
   - Assets in `dagster_orchestration/assets/` call `dbt seed`, `dbt run`, `dbt test` via subprocess
   - `definitions.py` defines 3 jobs with daily schedules (0 0 * * *)
   - Uses `DuckDBPandasIOManager` for pandas data persistence

2. **dbt projects** (`liquor_sales/`, `dbt_demo/`): Star schema transformations
   - `liquor_sales` is the primary example with fact/dimension tables
   - Models reference snapshots (`item_snapshot`, `store_snapshot`) for slowly-changing dimensions
   - Tests configured in `schema.yml` files at column and model levels

3. **Great Expectations** (referenced in `GX_lessons.ipynb`): Data quality validation
   - Python-based validation framework for files, databases, and data lakes

## Key Conventions & Patterns

### Dagster Asset Dependencies
- Assets are linked via function parameters: `pipeline_one_asset_b(pipeline_one_asset_a: pd.DataFrame)`
- Explicit `@asset(deps=[])` used for subprocess-based assets (dbt) that don't return data
- Job definitions use `AssetSelection` to group related assets

### dbt Structure
- **Schema organization**: `models/raw/` (staging), `models/pipelines/` (transformed), `models/star/` (dimensional)
- **Snapshots**: Located in `snapshots/` with naming pattern `{entity}_snapshot.sql`
- **Tests**: Column-level tests in `schema.yml` alongside model definitions
- **Materialization**: Set globally in `dbt_project.yml` with `+materialized: table`

### Data Quality Testing (Pattern from `liquor_sales`)
- **Column tests**: `unique`, `not_null` defined directly in `schema.yml`
- **Reference tests**: Foreign key relationships via `relationships` test
- **dbt_utils tests**: `accepted_range` (numeric bounds), `expression_is_true` (row validation)
- **dbt_expectations tests**: Type validation with `expect_column_values_to_be_of_type`
- Tests run after `dbt run` as separate materialization stage

## Development Workflows

### Activate Conda Environment
```bash
conda activate elt          # For dbt, Meltano, Great Expectations
conda activate dagster      # For Dagster orchestration
```

### dbt Workflow (from `extra/liquor_sales`)
```bash
dbt debug                   # Validate BigQuery connection
dbt seed                    # Load CSV seeds into database
dbt snapshot                # Create SCD Type 2 snapshots
dbt run                     # Execute model transformations
dbt test                    # Run all data quality tests
dbt run --select fact_sales # Target specific model
dbt test --select fact_sales # Test specific model
```

### Dagster Development
```bash
cd extra/dagster_orchestration_dbt
# Set up .env with GITHUB_TOKEN and gcloud authentication
DAGSTER_DBT_PARSE_PROJECT_ON_LOAD=1 dagster dev
```
- Runs on `http://localhost:3000`
- UI shows asset graph, execution status, logs
- "Materialize selected" button manually triggers asset jobs
- Toggle scheduler to enable cron-based execution

### Great Expectations (from `GX_lessons.ipynb`)
```bash
conda activate elt
jupyter notebook GX_lessons.ipynb
```

## Critical Integration Points

### BigQuery Configuration
- dbt requires `profiles.yml` with GCP project ID in connection config
- `extra/dagster_orchestration_dbt/profiles.yml` must specify `project` and `keyfile` path
- Run `gcloud auth application-default login` for local development authentication

### Environment Variables
- Dagster requires `.env` file in `extra/dagster_orchestration_dbt/` with `GITHUB_TOKEN` for Git integration
- dbt seeds reference CSV files: `extra/liquor_sales/seeds/raw_customers.csv`

### Cross-Component Communication
- Dagster calls dbt via subprocess with current working directory context
- dbt snapshots track history; models reference snapshots with `{{ ref('item_snapshot') }}`
- Filter logic applies to SCD: `WHERE CURRENT_TIMESTAMP > dbt_valid_from and dbt_valid_to IS NULL`

## Assignment Pattern

Student tasks involve:
1. Adding column-level tests from `dbt_utils` (e.g., `accepted_range`) to `fact_sales`
2. Adding type validation tests from `dbt-expectations` to fact/dimension tables
3. Understanding test failure scenarios and debugging via dbt logs and SQL queries

Test additions follow this YAML pattern:
```yaml
- name: column_name
  tests:
    - dbt_utils.accepted_range:
        min_value: "value"
        max_value: "value"
    - dbt_expectations.expect_column_values_to_be_of_type:
        column_type: string
```

## Key Files to Reference

- `lesson.md`: Complete setup and execution instructions
- `extra/liquor_sales/models/schema.yml`: Example test patterns
- `extra/liquor_sales/models/star/dim_item.sql`: SCD Type 2 snapshot usage
- `extra/dagster_orchestration_dbt/dagster_orchestration/definitions.py`: 3-job orchestration pattern
- `GX_lessons.ipynb`: Great Expectations workflow
