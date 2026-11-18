# Niagara Syslog Processing

A development stack with Grafana, Loki, Tempo, Prometheus, and an OpenTelemetry
Collector in `grafana/otel-lgtm`, plus Grafana Alloy for syslog ingestion.

```text
Syslog devices -> Alloy (TCP/UDP) -> Loki -> Grafana Explore
Applications   -> OTLP Collector -> Loki / Tempo / Prometheus
```

## Start

Use Docker with Compose, or Podman with a Compose provider such as
`podman-compose`. On macOS, start your Podman machine first if it is stopped.

```sh
git clone https://github.com/ACE-IoT-Solutions/niagara-syslog-processing.git
cd niagara-syslog-processing
cp .env.example .env
# Edit .env if any ports are already occupied (for example GRAFANA_PORT=3300).
podman compose up -d
podman compose ps
podman compose logs -f alloy
```

Use `docker compose` in place of `podman compose` for Docker. The first start
downloads the images and waits for Grafana, Loki, and Prometheus to be ready
before starting Alloy.
The images use `latest`; pin their tags or digests if you need repeatable versions.

## Endpoints

| Service | Default host endpoint | Notes |
| --- | --- | --- |
| Grafana | http://localhost:3000 | Login: `admin` / `admin` |
| OTLP gRPC | `HOST:4317` | OpenTelemetry telemetry |
| OTLP HTTP | `http://HOST:4318` | Standard `/v1/logs`, `/v1/metrics`, `/v1/traces` paths |
| Prometheus | http://localhost:9090 | Metrics UI |
| RFC3164 syslog | `HOST:1514`, TCP and UDP | Legacy BSD syslog |
| RFC5424 syslog | `HOST:1515`, TCP and UDP | Structured syslog |
| Alloy | http://localhost:12345 | Pipeline status, `/metrics`, `/-/ready` |

`HOST` is this machine's reachable IP address. Syslog and OTLP bind to all host
interfaces. Syslog is open plaintext with no authentication. Grafana, Prometheus,
and Alloy's UI bind to localhost. This setup is intended for development on a
trusted network, consistent with the [LGTM image's intended use](https://github.com/grafana/docker-otel-lgtm).

Configure your Niagara station or other sender with this host, the matching
syslog format's port, and TCP or UDP. These listeners require messages that
conform to their selected RFC; they do not auto-detect formats or accept arbitrary
plain text. TCP senders can use newline-delimited or RFC6587 octet-counted framing.
For a device fixed to port 514, set `SYSLOG_PORT=514` in `.env` to map host port
514 to Alloy's RFC3164 listener. Rootless Linux runtimes may restrict ports below
1024; the default 1514 avoids that restriction.

## Send sample logs

Once Alloy is running, this Python 3 snippet sends a message through each of the
four listeners. Adjust the ports if you changed `.env`.

```sh
python3 - <<'PY'
import datetime
import socket

now = datetime.datetime.now(datetime.timezone.utc)
messages = {
    1514: f"<134>{now.strftime('%b %d %H:%M:%S')} test-station niagara: syslog smoke test",
    1515: f"<134>1 {now.isoformat(timespec='seconds')} test-station niagara - - - syslog smoke test",
}
for port, message in messages.items():
    for transport in (socket.SOCK_DGRAM, socket.SOCK_STREAM):
        with socket.socket(socket.AF_INET, transport) as sock:
            sock.settimeout(5)
            sock.connect(('127.0.0.1', port))
            sock.sendall((message + '\n').encode())
        print(f'Sent to {port} via {"UDP" if transport == socket.SOCK_DGRAM else "TCP"}')
PY
```

In Grafana, open **Explore**, select **Loki**, and query:

```logql
{job="syslog"}
```

Narrow the query with labels such as `hostname="test-station"`, `app="niagara"`,
`format="rfc3164"`, or `transport="udp"`. Alloy also preserves severity and
facility as labels. Logs use receipt time so inaccurate device clocks do not hide
new entries. The [Alloy syslog documentation](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.syslog/)
describes the supported formats and metadata.

## Manage

```sh
podman compose logs -f lgtm alloy
podman compose restart alloy  # After editing alloy/config.alloy
podman compose down           # Stop and remove containers; retain data
```

The `lgtm-data` volume persists LGTM data under `/data`; `alloy-data` stores Alloy
state. The default Loki writer buffers in memory and retries failed requests;
this simple setup does not provide durable syslog delivery across Alloy restarts,
and UDP has no delivery guarantee. `ENABLE_LOGS_ALL=true` enables diagnostic
output from LGTM's own services; Alloy supplies the syslog input separately.

To deliberately delete stored data as well, run `podman compose down -v`.

### Podman VM disk space

Loki can report healthy while refusing writes if the filesystem holding its WAL
exceeds the [default 90% disk usage threshold](https://grafana.com/docs/loki/latest/configure/#ingester).
Check with `podman compose exec lgtm df -h /data`.

On macOS, if the host has space but the Podman VM disk is full, set
`LGTM_DATA_PATH=./.data/lgtm` in `.env` and create that directory before starting.
This stores LGTM data on the shared host filesystem instead of the VM's named
volume. Changing this setting switches data stores; copy existing `/data`
contents into the new directory while LGTM is stopped if you want to preserve
them. Host directory data is not removed by `compose down -v`.

## License

This project's Compose configuration and Alloy configuration are available under
the [MIT License](LICENSE). The bundled container images and their components
retain their respective upstream licenses.
