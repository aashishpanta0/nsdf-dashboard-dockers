# NSDF Dashboard Containers

The image clones `sci-visus/openvisuspy` at `test_region_extraction` and
serves its Panel dashboard. `PYTHONPATH=/opt/openvisuspy/src` makes that
checkout importable without editing `app/main.py` or installing a different
release of `openvisuspy` from PyPI.

The image uses Ubuntu 24.04 with Python 3.12 in `/opt/venv` and installs `bokeh`, `panel`, `ipywidgets`,
`OpenVisus`, `ipywidgets_bokeh`, `ipykernel`, `boto3`, `xmltodict`, `aiohttp`,
and `colorcet` with pip. Docker provides the isolated environment, so Conda
is not needed inside the container.

Compose defaults to Linux AMD64 because the tested Linux ARM64 OpenVisus
package lacks native bindings. Docker Desktop on Apple Silicon runs this
image with emulation. The build checks OpenVisus and dashboard imports.

## Launch

Install and start Docker Desktop (or Docker Engine with the Compose plugin).

```sh
cp .env.example .env
mkdir -p data
docker compose up --build -d
docker compose ps
```

By default, `COMPOSE_PROFILES` in `.env` enables all three dashboards:
LLC2160 at <http://localhost:11957/app>, NEX-GDDP-CMIP6 at
<http://localhost:12347/dashboards>, and Somospie at
<http://localhost:11657/app>. Set valid Somospie dataset credentials in
`.env` first. The first build installs the upstream
sources and dashboard dependencies and can take several minutes.
The LLC2160 Dashboard defaults to `json/nasa.json` for NASA salinity, ocean
velocity, and temperature. These are remote datasets, so rendering also
depends on those servers being reachable.

## Select Dashboards

`COMPOSE_PROFILES` in `.env` controls which dashboards Compose includes. The
sample enables all three. Remove any profile name to disable that dashboard;
for example, set it to `llc2160,nsdf-somospie` to omit NEX-GDDP-CMIP6. Then
run `docker compose up -d`. To stop a dashboard that is already running, run
`docker compose stop <service>` using `llc2160`, `nex-gddp-cmip6`, or
`nsdf-somospie`.

## Choose Data

Set `DASHBOARD_DATASET` in `.env` to a dashboard JSON file, an IDX file, or
a URL accepted by OpenVisus. Files placed in `./data` are available read-only
at `/data` inside the container. For example:

```dotenv
DASHBOARD_DATASET=/data/dashboards.json
```

Dashboard JSON references to local datasets must also use container paths,
such as `/data/dataset/visus.idx`, rather than host paths. The named
`visus-cache` volume persists downloaded data across container restarts.

After changing `.env`, run `docker compose up -d` to recreate the service.

## Deploy

Copy this project directory to a Docker-capable server, for example
`/opt/nsdf-dashboard-dockers`. Run Compose on that server from the directory
containing `docker-compose.yml`, not from inside the cloned `openvisuspy`
directory. The source checkout is created inside the image during the build.

Create `.env` from `.env.example` if it does not already exist. To expose
the service directly on port 11957, set:

```dotenv
DASHBOARD_BIND_ADDRESS=0.0.0.0
DASHBOARD_PORT=11957
BOKEH_ALLOW_WS_ORIGIN=*
```

Then run on the server:

```sh
cd /opt/nsdf-dashboard-dockers
mkdir -p data
docker compose up --build -d
docker compose ps
```

Open `http://SERVER_IP:11957/app` from your computer. Allow inbound TCP port
11957 in the server firewall and any hosting-provider firewall, preferably
only from trusted clients. The server also needs outbound HTTPS access to
GitHub and PyPI during the build and to the NASA dataset service on port
50098 at runtime.

The default `BOKEH_ALLOW_WS_ORIGIN=*` allows WebSocket connections from any
origin; it does not provide authentication or open the server firewall.
For public deployments, replace `*` with the browser-facing hostname and
port, such as `dashboard.example.org:11957`.

For an HTTPS reverse proxy, keep the loopback binding if the proxy runs on
the host, forward to `127.0.0.1:11957`, enable WebSocket forwarding, and set
`BOKEH_ALLOW_WS_ORIGIN=dashboard.example.org` to match the browser-facing
hostname. A containerized proxy needs shared-network routing instead of
the host loopback address. The dashboard has no authentication configured;
use TLS and authentication at the proxy before exposing private data.

The default tracks a mutable branch. For repeatable releases, set
`OPENVISUSPY_REF` to a fixed upstream tag and pin dependencies as needed.
This build argument accepts branch and tag names, not raw commit hashes.
Rebuild with `docker compose build --no-cache` when refreshing a branch
that has advanced; Docker can otherwise reuse the cached checkout.

## NEX-GDDP-CMIP6 Dashboard

`Dockerfile.nex-gddp-cmip6` clones the public `aashishpanta0/openvpy-cmip6-public`
repository at `main`, installs the supplied scientific and Jupyter library
list plus `OpenVisusNoGui`, and serves NEX-GDDP-CMIP6 over HTTP on port 12347.
It is independent of the NASA LLC2160 image and cache.

Set the `NEX_GDDP_CMIP6_*` dataset and port settings in `.env` as needed.
CMIP6 does not require TLS certificate files; access it locally at
`http://localhost:12347/dashboards`.

From this project's directory on the server, start just CMIP6:

```sh
docker compose up --build -d nex-gddp-cmip6
docker compose logs -f nex-gddp-cmip6
```

The container still needs outbound HTTPS access to the dataset server. To
start the other dashboards too, use the same `docker compose up` command shown
above.

Its restart policy replaces the infinite shell loop.
The requested `--dev` and wildcard WebSocket origin are retained; remove
`--dev` and restrict origins for production. No authentication is configured.
This dependency list is substantially larger than the first image and can
take several minutes to build. `NEX_GDDP_CMIP6_REF` accepts upstream branches or tags;
use `docker compose build --no-cache nex-gddp-cmip6` to refresh a cached branch.

## NSDF Somospie

Somospie is independently selectable through `COMPOSE_PROFILES` and uses its
own `Dockerfile.nsdf-somospie`, which checks out the `nsdf-ahm` branch by
default. It serves the branch's dashboards over HTTP on localhost port 11657
and has no certificate mount. Set `SOMOSPIE_DATASET` in `.env` to the Wasabi
IDX URL with valid credentials. Keep access and secret keys private; `.env`
is git-ignored.

Start or inspect Somospie through the shared Compose file:

```sh
docker compose up --build -d
docker compose logs -f nsdf-somospie
```

Open `http://localhost:11657/dashboards`.

## Operations

```sh
docker compose config
docker compose ps
docker compose logs --tail=100 llc2160
docker compose down
```

`docker compose down` preserves the cache. Adding `--volumes` deletes it.