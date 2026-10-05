# NSDF Dashboard Containers

Docker Compose runs three dashboards:

| Service | URL | Dataset setting |
| --- | --- | --- |
| LLC2160 | <http://localhost:11957/app> | `LLC2160_DATASET` |
| NEX-GDDP-CMIP6 | <http://localhost:12347/dashboards> | `NEX_GDDP_CMIP6_DATASET` |
| Somospie | <http://localhost:11657/dashboards> | `SOMOSPIE_DATASET` |

## Requirements

Install and start Docker Desktop or Docker Engine with the Compose plugin.
On Apple Silicon, the images run as Linux AMD64 under emulation.

## First Run

Run these commands from the project directory:

```sh
cp -n .env.example .env
```

Set valid Wasabi dataset credentials in `SOMOSPIE_DATASET` in `.env` before
starting Somospie. The `.env` file is git-ignored. Local LLC2160 data can be
placed in `data/` and referenced inside the container under `/data`. Set
`LLC2160_DATASET` to choose the LLC2160 dataset; it defaults to
`json/nasa.json`. For a local dataset, use its container path, such as
`/data/my-dataset/visus.idx`.

## Start Dashboards

Start all three dashboards:

```sh
docker compose up --build -d
```

Start only one dashboard by naming its Compose service:

```sh
docker compose up --build -d llc2160
docker compose up --build -d nex-gddp-cmip6
docker compose up --build -d nsdf-somospie
```

The sample `.env` enables all three services for the command that starts all
dashboards. The individual commands start only the named service. The first
build downloads the dashboard sources and dependencies and may take several
minutes.

## Stop and Inspect

```sh
docker compose ps
docker compose logs -f <service>
docker compose stop <service>
docker compose down
```

Use `llc2160`, `nex-gddp-cmip6`, or `nsdf-somospie` for `<service>`.
`docker compose down` preserves downloaded caches; add `--volumes` to delete
them.