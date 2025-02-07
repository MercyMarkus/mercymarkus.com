---
title: 'Data Engineering Zoomcamp by DataTalks.Club: Module 2'
date: 2025-02-05T12:06:03+01:00
draft: false
description: "Introduction to Data Lakes and Workflow Orchestration. Workflow orchestration with Kestra"
series: ['DTC-DEZ']
tags: [Data, Cloud, Docker, Kestra]
---


This module covers workflow orchestration using [Kestra](https://go.kestra.io/de-zoomcamp/github).

Kestra is an open-source, event-driven orchestration platform that simplifies building both scheduled and event-driven workflows. By adopting Infrastructure as Code practices for data and process orchestration, Kestra enables you to build reliable workflows with just a few lines of YAML.

> [!NOTE]
> The files I reference/used for the course are hosted in this [git repository](https://github.com/MercyMarkus/2025_zoomcamp/tree/main)


> [!NOTE]
>You can find all videos for this module in this [YouTube Playlist](https://go.kestra.io/de-zoomcamp/yt-playlist).

## Build Data Pipelines with Kestra

This module involved building ETL pipelines for Yellow and Green Taxi data from NYC’s Taxi and Limousine Commission (TLC) for the years 2019 & 2020 using Kestra both locally and in GCP.

### Learning Points

1. Extracting data from [CSV files](https://github.com/DataTalksClub/nyc-tlc-data/releases).
2. Loading it into Postgres locally and on Google Cloud (GCS + BigQuery).
3. Exploring scheduling and backfilling Kestra workflows both locally and on Google Cloud (GCS + BigQuery).

### Notes

Following along to this module was mostly straight forward with a few gotchas. Some of these were:

- Running Kestra, Postgres and PgAdmin using Docker and accessing the Kestra & PgAdmin Dashboard locally didn't work with the `docker-compose.yml` provided for the module.
  - This was partly because Kestra and PgAdmin's default ports are both on port 8080 and needed to be updated. Thankfully, the [module's FAQs](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/02-workflow-orchestration#troubleshooting-tips) came in handy and provided a sample Docker Compose file with guidance on resolving this and the **Connection Refused** errors for linux users.
- Linux users will encounter **Connection Refused** errors when connecting to the Postgres DB from within Kestra. This is because `host.docker.internal` works differently on Linux.
  - Updating the pluginDefaults connection info for the different flows referencing it (except for the `03_postgres_dbt.yaml` flow) to the name of the Postgres image defined in the `docker-compose.yml` file resolved this.

#### Previous

```yaml
pluginDefaults:
  - type: io.kestra.plugin.jdbc.postgresql
    values:
      url: jdbc:postgresql://host.docker.internal:5432/postgres-zoomcamp
      username: kestra
      password: k3str4
```

#### Updated

```yaml
pluginDefaults:
  - type: io.kestra.plugin.jdbc.postgresql
    values:
      url: jdbc:postgresql://postgres_zoomcamp:5432/postgres-zoomcamp
      username: kestra
      password: k3str4
```

- The fix for the `03_postgres_dbt.yaml` connection error was slightly different and required the hostname of my local computer defined in the file i.e. `host` variable. I got this by running the `hostname -I` command in a terminal and selecting the first IP.

> If you prefer a GUI, you can also find it through the overview tab of the **Systems Monitor** (on `Fedora` it's in the **Network & Systems** Chart) or under Network **Settings** on `Ubuntu`.

#### Previous Block

```yaml
profiles: |
      default:
        outputs:
          dev:
            type: postgres
            host: host.docker.internal
            user: kestra
            password: k3str4
            port: 5432
            dbname: postgres-zoomcamp
            schema: public
            threads: 8
            connect_timeout: 10
            priority: interactive
```

#### Updated Block

```yaml
profiles: |
      default:
        outputs:
          dev:
            type: postgres
            host: 192.169.0.23
            user: kestra
            password: k3str4
            port: 5432
            dbname: postgres-zoomcamp
            schema: public
            threads: 8
            connect_timeout: 10
            priority: interactive
        target: dev
  ```

  **P.S**: There's an [updated fix](https://github.com/DataTalksClub/data-engineering-zoomcamp/commit/25ce6aa101d5f7f1198c790199dbe5723b2ee5a0) for this in the dbt flow.
