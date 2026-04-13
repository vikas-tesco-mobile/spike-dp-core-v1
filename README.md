# spike-dp-core (Approach 1 - Split Bundles)

## Approach

Both repos have their own Databricks Asset Bundle and deploy independently. This repo (core) builds the wheel in CD, owns Spark source code, configs, schemas, and deploys bronze/silver workflows. The analysts repo owns the dbt project and deploys only dbt workflows. The cross-repo contract is the **silver tables** -- dbt workflows in the analysts repo trigger when silver tables are updated by this repo.

## What This Repo Contains

| Component | Description |
|---|---|
| `src/dp_core/` | Spark Python code -- extractors, loaders, transformers, exporters, housekeeping, utilities |
| `configs/envs/` | Jinja2 config template (`config.yml.j2`) and `.env.template` for environment-specific values |
| `configs/schemas/` | Silver layer schema definitions (12 YAML files) |
| `workflows/` | 10 Databricks workflow YAMLs (bronze, silver, exporters, housekeeping) |
| `tests/` | Python unit tests (pytest) |
| `databricks.yml` | Databricks Asset Bundle config (bundle name: `dp-core`) |
| `pyproject.toml` | Python project config (package name: `dp-core`, internal: `dp_core`) |

## Workflows Owned

### Finance
- `tesco_locations_api_daily` -- extract API -> bronze -> silver
- `tesco_products_api_daily` -- extract API -> bronze -> silver
- `tesco_quote_stream_daily` -- bronze only
- `tesco_tap_store_online_sales` -- bronze -> silver (store + online)
- `sale_transactions_exporter` -- gold -> CSV export

### CCS / Revenue Assurance
- `vmo2_mft_usage_daily` -- bronze -> 8 silver tables
- `offer_manager_daily` -- 3 bronze tables
- `hansen_daily` -- 5 bronze tables
- `blugem_ccs_usage_reconciliation_exporter` -- gold -> CSV export

### Housekeeping
- `table_metadata_updater` -- metadata tags and descriptions

## CI/CD

| Pipeline | What It Does |
|---|---|
| `ci.yml` | Snyk vulnerability scan, pre-commit (ruff, mypy), pytest with coverage |
| `cd.yml` | Builds wheel, deploys to dev via `databricks bundle deploy` |

## Local Development

```bash
poetry install --with dev
cp configs/envs/.env.template configs/envs/.env
# Edit .env with your DEVELOPER_PREFIX
databricks bundle deploy --target local
```

## Relationship With spike-dp-analysts

```
spike-dp-core                          spike-dp-analysts
(this repo)                            (separate repo)
 Bronze/Silver workflows                dbt workflows
 Spark code + wheel                     dbt models (gold layer)
 configs + schemas                      profiles.yml
         |                                     |
         v                                     v
   Silver tables  ---- trigger ---->   dbt builds gold tables
```

## Observations

- Responsibilities are clearly split. This repo owns ingestion, bronze/silver processing, schemas, and operational workflows, while analysts owns dbt and gold-layer workflows.
- The integration contract is the silver layer. That gives this repo strong ownership of upstream data structure and refresh behavior.
- Deployment is self-contained for core. The repo can build its wheel and deploy its own bundle without waiting for analysts bundle changes.
- This approach works well when the main reuse mechanism is shared datasets rather than shared Python execution across repos.
- Operational ownership is easier to explain: if the issue is in bronze or silver generation, it belongs here; if it is in dbt gold modeling, it belongs in analysts.

## Limitations

- Downstream compatibility is not version-pinned. A change to silver schemas or semantics can affect analysts immediately after deployment if the contract is not carefully managed.
- This repo still builds a wheel, but that wheel is not the primary cross-repo contract in approach 1, so some packaging effort provides less integration value than in a library-based approach.
- End-to-end validation requires coordination with analysts because local success in this repo does not prove that downstream dbt workflows still work.
- Data contract changes need stronger discipline around schema evolution, backward compatibility, and deployment sequencing.
- Compared with a shared-library model, it is harder to make code-level reuse explicit because the repos integrate mainly through tables and workflow triggers.
