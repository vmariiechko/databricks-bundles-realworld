# Databricks Multi-Environment Deployment Template

A production-ready template for deploying Databricks Asset Bundles across multiple environments (user, dev, stage, prod) with environment-specific configurations.

**Focus:** This template demonstrates deployment patterns and environment management, not specific business logic. The sample code is intentionally minimal—replace it with your actual workloads.

## What This Template Provides

- **Multi-environment strategy**: Isolated deployments for user, dev, stage, and prod
- **Environment-specific overrides**: Different compute, schedules, and schemas per environment
- **Serverless-first**: Ready for Databricks Free Edition
- **Modular structure**: Clean separation between resources and code
- **CI/CD ready**: Git-based deployment workflows
- **Permissions**: Comprehensive RBAC for schemas and resources

## Project Structure

```
.
├── databricks.yml              # Bundle configuration with 4 environments
├── variables.yml               # Shared variables
├── resources/
│   ├── sample_ingestion.job.yml         # Sample: Multi-task ingestion job
│   ├── sample_pipeline.pipeline.yml     # Sample: LDP pipeline
│   ├── sample_pipeline_trigger.job.yml  # Sample: Pipeline trigger job
│   └── medalion_architecture.schemas.yml # Schema definitions
├── templates/
│   ├── cluster_configs.yml     # Copy-paste cluster configurations
│   └── README.md               # Templates usage guide
├── src/
│   ├── jobs/
│   │   ├── ingest_to_raw.py    # Sample ingestion code
│   │   └── transform_to_silver.py # Sample transformation code
│   └── pipelines/
│       ├── bronze.py           # Sample LDP bronze layer
│       └── silver.py           # Sample LDP silver layer
└── docs/
    ├── SETUP_GROUPS.md         # Group creation guide
    ├── PERMISSIONS_SETUP.md    # Permissions setup guide
```

## Quick Start

### 1. Prerequisites

- Databricks workspace (Free Edition or paid)
- Databricks CLI installed: `pip install databricks-cli`
- Unity Catalog with a catalog created

### 2. Deploy to Your Workspace

```bash
# Validate the bundle
databricks bundle validate -t user

# Deploy to your user environment (isolated, development mode)
databricks bundle deploy -t user

# Run the sample job
databricks bundle run sample_ingestion -t user

# Trigger the sample pipeline
databricks bundle run sample_pipeline_trigger -t user
```

### 3. Verify Deployment

Check your Databricks workspace:
- **Jobs**: Look for `[user <yourname>] Sample Ingestion Job`
- **Pipelines**: Look for `[user <yourname>] Sample ETL Pipeline`

## Environment Strategy

### Four Deployment Targets

| Target | Purpose | Mode | Naming | Schedule | Compute |
|--------|---------|------|--------|----------|---------|
| **user** | Local development | development | `[user <name>]` prefix | Paused | Serverless |
| **dev** | Shared development | production | `[dev]` prefix | Active | Configurable |
| **stage** | Pre-production | production | `[stage]` prefix | Active | Production-like |
| **prod** | Production | production | No prefix | Active | Production |

### Key Differences Per Environment

```yaml
# User: Fast iteration, isolated
user:
  mode: development
  presets:
    name_prefix: "[user ${workspace.current_user.short_name}] "
    trigger_pause_status: PAUSED
    pipelines_development: true

# Dev: Shared team environment
dev:
  mode: production
  git:
    branch: dev
  # Resources optimized for cost

# Stage: Production mirror
stage:
  mode: production
  git:
    branch: stage
  run_as:
    service_principal_name: "<spn-id>"
  # Full production configuration

# Prod: Production
prod:
  mode: production
  git:
    branch: main
  run_as:
    service_principal_name: "<spn-id>"
  # Maximum reliability and performance
```

## Permissions and Access Control

This template includes comprehensive permissions configuration for managing secure, role-based access.

### Quick Start

**Option 1: Just Testing** (No Setup Required)
```bash
databricks bundle deploy -t user  # Automatic full access in development mode
```

**Option 2: Team Collaboration** (5-Minute Setup)
1. Create required groups in Databricks workspace (see [docs/SETUP_GROUPS.md](./docs/SETUP_GROUPS.md))
2. Deploy: `databricks bundle deploy -t dev`

**Option 3: Production Ready** (Full Setup)
1. Create service principals
2. Configure in `variables.yml`
3. Enable in `databricks.yml`

### Documentation

| Document | Purpose |
|----------|---------|
| **[docs/SETUP_GROUPS.md](./docs/SETUP_GROUPS.md)** | How to create required groups |
| **[docs/PERMISSIONS_SETUP.md](./docs/PERMISSIONS_SETUP.md)** | Complete setup guide |

## Compute Options

This template supports both **serverless** and **custom cluster** compute:

### Serverless (Default)
- No cluster management needed
- Works on Databricks Free Edition
- Configured via `environments` section in jobs

### Custom Clusters
- Full control over cluster specs
- Better for specific workload requirements
- Templates available in `templates/cluster_configs.yml`

To switch to custom clusters:
1. Comment out `environments` section
2. Uncomment `job_clusters` section
3. Update task `job_cluster_key` references

## Customization

### Replace Sample Code

The `src/` directory contains minimal example code. **Replace it with your actual code:**

1. **Jobs** (`src/jobs/*.py`): Replace with your ingestion/transformation logic
2. **Pipelines** (`src/pipelines/*.py`): Replace with your LDP transformations

### Add Resources

Create new resource files in `resources/`:

```yaml
# resources/my_new_job.job.yml
resources:
  jobs:
    my_new_job:
      name: "My New Job"
      tasks:
        - task_key: main
          environment_key: default
          spark_python_task:
            python_file: "${workspace.file_path}/src/my_script.py"
```

### Environment-Specific Overrides

Override any resource setting per environment in `databricks.yml`:

```yaml
targets:
  prod:
    resources:
      pipelines:
        sample_pipeline:
          schema: "prod_schema"
          continuous: true
      jobs:
        sample_ingestion:
          timeout_seconds: 14400  # 4 hours
```

## Sample Code Overview

The included code is intentionally simple to demonstrate the pattern:

### Ingestion Job (`src/jobs/ingest_to_raw.py`)
- Reads from `samples.bakehouse.sales_customers` (built-in Databricks sample data)
- Writes to bronze schema
- **Replace with**: Your actual data ingestion logic

### Transform Job (`src/jobs/transform_to_silver.py`)
- Reads from bronze layer
- Applies basic cleaning and writes to silver
- **Replace with**: Your transformation logic

### LDP Bronze Layer (`src/pipelines/bronze.py`)
- Creates `taxi_trips_raw` table from sample data
- Adds ingestion timestamp
- **Replace with**: Your bronze layer transformations

### LDP Silver Layer (`src/pipelines/silver.py`)
- Aggregates bronze data by pickup zone
- **Replace with**: Your silver layer business logic

## CI/CD Workflow

### Recommended Git Workflow

```
feature/my-work → dev branch → stage branch → main branch
     ↓               ↓             ↓             ↓
   (local)      dev target    stage target   prod target
```

### Example GitHub Actions

```yaml
name: Deploy to Dev
on:
  push:
    branches: [dev]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy bundle
        run: |
          pip install databricks-cli
          databricks bundle deploy -t dev
        env:
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
```

## Configuration Variables

Edit `variables.yml` for environment-agnostic settings:

```yaml
variables:
  catalog_name:
    description: Unity Catalog name
    default: "my_catalog"

  failure_notification_emails:
    description: Email addresses for job failure alerts
    default: []

  pipeline_min_workers:
    description: Min workers for pipeline autoscale
    default: 1

  photon_enabled:
    description: Enable Photon acceleration
    default: true
```

## Free Edition Notes

The template works on Databricks Free Edition:

- ✅ Uses serverless compute by default
- ✅ Jobs and pipelines supported
- ✅ No custom clusters required
- ⚠️ Limited to 1 active pipeline per type

To use custom clusters (paid workspaces), see `templates/cluster_configs.yml` for configurations.

## Testing

```bash
# Validate configuration
databricks bundle validate -t user

# Deploy and test
databricks bundle deploy -t user
databricks bundle run sample_ingestion -t user

# Clean up
databricks bundle destroy -t user
```

## What to Modify for Your Use Case

1. **Bundle name** (`databricks.yml`): Change `realworld_example` to your project name
2. **Catalog name** (`variables.yml` and targets): Update to your Unity Catalog
3. **Sample code** (`src/`): Replace with your actual business logic
4. **Resource files** (`resources/`): Add/modify jobs and pipelines
5. **Workspace hosts** (`databricks.yml`): Update to your workspace URLs
6. **Service principals** (`databricks.yml`): Configure for stage/prod (optional)
7. **Permissions** (`databricks.yml`): Customize access control for your organization

## Additional Resources

- [Databricks Asset Bundles Documentation](https://docs.databricks.com/dev-tools/bundles/)
- [Deployment Modes](https://docs.databricks.com/dev-tools/bundles/deployment-modes.html)
- [Lakeflow Declarative Pipelines](https://docs.databricks.com/delta-live-tables/)
- [Serverless Compute](https://docs.databricks.com/serverless-compute/)

## Contributing

This is a template repository. Fork it, customize it, and make it your own!

## License

This template is provided as-is for use in your Databricks projects.
