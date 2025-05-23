---
title: 'Data Engineering Zoomcamp by DataTalks.Club: Module 1'
date: 2025-01-07T12:06:03+01:00
draft: false
description: "Learn the basics of GCP, Docker, Docker Compose, running PostgreSQL in Docker, and cloud infrastructure setup with Terraform."
series: ['DTC-DEZ']
tags: [Data, Cloud, Docker]
---

After hours of bumbling about, I finally understood the structure of the self-paced course: the follow-along videos are hosted on DTC's YouTube channel, while the notes on GitHub serve as supplementary material rather than the primary source.

P.S: I'm on a Linux (Fedora 40) machine and working from a [virtual environment](https://www.freecodecamp.org/news/how-to-setup-virtual-environments-in-python/). I also created a folder for the course where I save the files. All commands are run from that folder.

Docker Installation Instructions for Fedora are [here](https://docs.docker.com/engine/install/fedora/).

> [!NOTE]
> The files I reference/used for the course are hosted in this [git repository](https://github.com/MercyMarkus/2025_zoomcamp/tree/main).

{{< toc >}}

## [Module 1: Containerization and Infrastructure as Code](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/01-docker-terraform)

- Introduction to GCP
- Docker and Docker Compose
- Running PostgreSQL with Docker
- Infrastructure setup with Terraform
- Homework

### [DE Zoomcamp 1.2.1 - Introduction to Docker](https://www.youtube.com/watch?v=EYNwNlOrpr0&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=4)

| Description    | Command |
| -------------- | ------- |
| Run the Ubuntu container and initialize the Bash Shell. (`-it` means it opens a terminal with an interactive shell when running the Ubuntu container) | `docker run -it ubuntu bash`    |
| Run the python v3.9 container  | `docker run -it python:3.9`    |
| Run the python container with a bash entrypoint so you can install pandas onto the container  | `docker run -it --entrypoint=bash python:3.9`    |
| Build the [Dockerfile](https://github.com/MercyMarkus/2025_zoomcamp/blob/main/module_1/Dockerfile) from the pwd. Said file uses the Python 3.9 image, installs Pandas using pip and sets Bash as the entry point of the container. | `docker build -t test:pandas .` (test is the image name) |
| Run container built from the Dockerfile | `docker run -it test:pandas` |
| After updating the Dockerfile's entrypoint to `[ "python" , "pipeline.py"]`, you can pass the python file the date arg directly when running the container | `docker run -it test:pandas 2023-05-08` |

### [DE Zoomcamp 1.2.2 - Ingesting NY Taxi Data to Postgres](https://www.youtube.com/watch?v=2JM-ziJt0WI&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=5)

| Description | Command |
| ----------- | ------- |
| List the running docker containers | `docker ps` |
| Run PostgreSQL container with the necessary environment variables (`-e`), mounted volume (`-v`) & port mapping (`-p`)  | ``` docker run -it -e POSTGRES_USER="root" -e POSTGRES_PASSWORD="root" -e POSTGRES_DB="ny_taxi" -v $(pwd)/ny_taxi_postgres_data:/var/lib/postgresql/data -p 5432:5432 postgres:13 ``` |
| Adjust the permissions of the `ny_taxi_postgres_data` folder. _In Linux, the folder doesn't show up in VS Code and you won't have access to it in your File Manager_. | `sudo chmod a+rwx ny_taxi_postgres_data` |
| Install `pgcli` using pip. This didn't work out of the box for me and I had to manually install the `psycopg-binary` library using pip afterwards | `pip install pgcli psycopg-binary` |
| Connect to the database. _It won't have any data on it yet_ | `pgcli -h localhost -p 5432 -u root -d ny_taxi` |
| Optional: Install Jupyter to have access to Jupyter Notebooks; an interactive python editor | `pip install jupyter` |
| Open Jupyter| `jupyter notebook` (_create a new Python 3 file when it opens_) |
| Download NYC Taxi Data | `wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz` |
| Decompress NYC Taxi Data using Gzip | `gzip -d yellow_tripdata_2021-01.csv.gz` |
| View first 100 lines of the CSV file in the terminal | `less yellow_tripdata_2021-01.csv` or `head -n 100 yellow_tripdata_2021-01.csv` |
| Split CSV file into first 100 records | `head -n 100 yellow_tripdata_2021-01.csv > yellow_tripdata_head_100.csv` |
| Count number of records in the file | `wc -l yellow_tripdata_2021-01.csv` |
| Install `sqlalchemy` & `psycopg2-binary` using pip. I had to install `psycopg2-binary` because of the failed automatic pip installation from earlier. | `pip install sqlalchemy psycopg2-binary` |
| List PostgreSQL tables | In pgcli: `\dt` |
| View specific table information | In pgcli: `\d yellow_taxi_data` |

### [DE Zoomcamp 1.2.3 - Connecting pgAdmin and Postgres](https://www.youtube.com/watch?v=hCAIVe9N0ow&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=7)

| Description | Command |
| ----------- | ------- |
| Run pgAdmin as a Docker container | `docker run -it   -e PGADMIN_DEFAULT_EMAIL="admin@admin.com"   -e PGADMIN_DEFAULT_PASSWORD="root"   -p 8080:80   dpage/pgadmin4` |
| Create a network | `docker network create pg-network` |
| Run Postgres container with just created `pg-network`. The container name i.e. `pg-database` will be used as the Host name/address in the pgAdmin connection settings. | `docker run -it -e POSTGRES_USER="root" -e POSTGRES_PASSWORD="root" -e POSTGRES_DB="ny_taxi" -v $(pwd)/ny_taxi_postgres_data:/var/lib/postgresql/data -p 5432:5432 --network=pg-network --name pg-database postgres:13` |
| Run pgAdmin as a container with just created `pg-network` | `docker run -it   -e PGADMIN_DEFAULT_EMAIL="admin@admin.com"   -e PGADMIN_DEFAULT_PASSWORD="root"   -p 8080:80 --network=pg-network --name pgadmin-2 dpage/pgadmin` |

### [DE Zoomcamp 1.2.4 - Dockerizing the Ingestion Script](https://www.youtube.com/watch?v=B1WwATwf-vY&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=8)

| Description | Command |
| ----------- | ------- |
| Convert Jupyter notebook to python script | `jupyter nbconvert --to=script PostgreSQL_Data_Injestion.ipynb` |
| Run the python script after conversion and cleanup | `python ingest_data.py --user=root --password=root --host=localhost --port=5432 --db=ny_taxi  --table_name=yellow_taxi_trips --url=${URL}` |

### [DE Zoomcamp 1.2.5 - Running Postgres and pgAdmin with Docker-Compose](https://www.youtube.com/watch?v=hKI6PkPhpa0&t=400s)

**Note**: On Fedora, the command is `docker compose` and was installed when I installed the Docker Engine following the instructions [here](https://docs.docker.com/engine/install/fedora/).

| A Youtube comment recommended using `docker compose start/stop` as opposed to `docker compose up/down` if you want your database settings persisted in PgAdmin. Else running `docker compose up` will delete the local host server you created from lesson **1.2.3**.

| Description | Command |
| ----------- | ------- |
| Start docker services defined in `docker-compose.yaml` | `docker compose start` |
| Stop docker services defined in `docker-compose.yaml` | `docker compose stop` |
| Create and start docker containers. Add the `-d` option to run in detached mode | `docker compose up` |
| Stop and remove docker containers & networks | `docker compose down` |

### [DE Zoomcamp 1.2.6 - SQL Refresher](https://www.youtube.com/watch?v=QEcps_iskgg&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=10)

> **Note**: Avoid using a SELECT inside the IN(). The IN() construct is more useful when you pass a list of values as "IN(1, 2, 100, 250)".
>
> If there is any NULL value returned by the SELECT inside the IN(), the result will be empty, the IN() will "fail" and it will cause a lot of confusion and errors. It is a very difficult thing to debug.

Use EXISTS instead:

```sql
WHERE NOT EXISTS (
SELECT 1
from zones z
where z."LocationID" = t."PULocationID")
```

### Terraform Install on Fedora

- `sudo dnf install -y dnf-plugins-core`
- `sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo`
- `sudo dnf -y install terraform`
