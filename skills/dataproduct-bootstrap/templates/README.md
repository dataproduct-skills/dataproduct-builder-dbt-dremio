# {{DATA_PRODUCT_NAME}}

dbt data product `{{DATA_PRODUCT_ID}}` on Dremio. Published to [Entropy Data](https://entropy-data.com).

## Install

Project Python deps (dbt, the `dbt-dremio` adapter, `openlineage-dbt`, `datacontract`, `entropy-data`):

```bash
uv sync
```

`uv sync` creates `.venv/` with everything from `pyproject.toml`'s `[dependency-groups].dev`. All invocations below use the venv via `uv run` — no activation needed.

## Configure

Copy `profiles.yml.example` to `~/.dbt/profiles.yml` (or merge it in). The profile authenticates with a Dremio personal access token (Dremio: **Account Settings → Personal Access Tokens**):

```bash
export DREMIO_PAT=<your-dremio-personal-access-token>
```

Set the Entropy Data API key for OpenLineage transport:

```bash
export OPENLINEAGE__TRANSPORT__AUTH__APIKEY=<your-entropy-data-api-key>
```

## Run

```bash
uv run dbt-ol run
uv run dbt test
```

Tables are written as Apache Iceberg tables to `{{CATALOG}}.{{FOLDER}}` in Dremio's catalog.

## Layout

```
models/
├── input_ports/      # external sources you read from
├── staging/          # 1:1 cleaned views over input ports
├── intermediate/     # joined / shaped views
└── output_ports/v1/  # published tables — one per output port
```

Output ports are versioned (`v1`, `v2`, ...). Each version directory contains the SQL models plus the ODCS data contract that governs the schema (`<contract-id>.odcs.yaml`). Cached input-port contracts live in `models/input_ports/` next to their `.source.yaml`.

## Contract tests

The data contract declares two servers: `production` (type `dremio`, where consumers query the data) and `iceberg` (Dremio's Iceberg REST catalog, which `datacontract test` reads the tables through). Run the contract test with the same token:

```bash
export DATACONTRACT_ICEBERG_TOKEN=$DREMIO_PAT
export DATACONTRACT_ICEBERG_PROPERTIES='{"oauth2-server-uri":"https://login.dremio.cloud/oauth/token","header.X-Iceberg-Access-Delegation":"vended-credentials"}'
uv run datacontract test models/output_ports/v1/<contract-file>.odcs.yaml --server iceberg
```

## Publishing

CI in `.github/workflows/data-product.yml` runs `dbt run`, `dbt test`, publishes the data product spec and contract to Entropy Data, and runs the data contract test.
