# StatsD Exporter Snap

[![Pull Request](https://github.com/canonical/statsd-exporter-snap/actions/workflows/pull-request.yaml/badge.svg)](https://github.com/canonical/statsd-exporter-snap/actions/workflows/pull-request.yaml)
[![Release Snap](https://github.com/canonical/statsd-exporter-snap/actions/workflows/release.yaml/badge.svg)](https://github.com/canonical/statsd-exporter-snap/actions/workflows/release.yaml)

A snap package for [Prometheus StatsD Exporter](https://github.com/prometheus/statsd_exporter).

## Overview

The StatsD exporter is a drop-in replacement for StatsD. It receives StatsD-style metrics and exports them as Prometheus metrics.

### Key Features

- Translates StatsD metrics to Prometheus metrics via configurable mapping rules
- Supports multiple tagging formats: Librato, InfluxDB, DogStatsD, and SignalFX-style
- Converts gauges, counters, timers, histograms, and distributions to Prometheus equivalents

### Default Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 9125 | UDP/TCP  | StatsD listener |
| 9102 | TCP      | Prometheus metrics endpoint |

## Installation

```bash
sudo snap install statsd-exporter
```

## Usage

### Running as a Service (Daemon)

The snap includes a daemon service that is disabled by default. To enable and start it:

```bash
sudo snap start --enable statsd-exporter
```

To stop the service:

```bash
sudo snap stop statsd-exporter
```

## Configuration

### Mapping Configuration

The daemon looks for mapping configuration in the following order:
1. `/etc/statsd-exporter/*.yml` or `/etc/statsd-exporter/*.yaml` (requires connecting the system-files plug)
2. Built-in default configuration at `$SNAP/etc/mapping.yml`

To use external configuration:

```bash
# Connect the system-files plug
sudo snap connect statsd-exporter:etc-statsd-exporter-config

# Create and configure the mapping file
sudo mkdir -p /etc/statsd-exporter
sudo nano /etc/statsd-exporter/mapping.yml

# Restart the service
sudo snap restart statsd-exporter
```

### Example Mapping Configuration

```yaml
defaults:
  observer_type: histogram
  histogram_options:
    native_histogram_bucket_factor: 1.1
    native_histogram_max_buckets: 256
  match_type: glob

mappings:
  - match: "test.dispatcher.*.*.*"
    name: "dispatcher_events_total"
    labels:
      processor: "$1"
      action: "$2"
      outcome: "$3"
```

For detailed mapping configuration options, see the [upstream documentation](https://github.com/prometheus/statsd_exporter#metric-mapping-and-configuration).

## Quick Start: Sending Metrics

Once the exporter is running, you can send StatsD metrics via UDP and view them as Prometheus metrics.

### Send metrics

```bash
# Using Python (most portable)
python3 -c "
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Counter: increment a value
sock.sendto(b'http_requests:1|c', ('localhost', 9125))

# Gauge: set a current value
sock.sendto(b'queue_depth:42|g', ('localhost', 9125))

# Timer: record a duration in milliseconds
sock.sendto(b'api_latency:150|ms', ('localhost', 9125))

sock.close()
"

# Or using netcat
echo "http_requests:1|c" | nc -u -w0 localhost 9125
```

### View metrics

```bash
curl -s http://localhost:9102/metrics | grep -E '^(http_requests|queue_depth|api_latency)'
```

Example output:

```
http_requests 1
queue_depth 42
api_latency_bucket{le="0.25"} 1
api_latency_bucket{le="+Inf"} 1
api_latency_sum 0.15
api_latency_count 1
```

Timers are converted to histograms (seconds) by default. Counters and gauges are exported directly.

## Development

This snap follows the [Canonical Observability snaps blueprint](https://github.com/canonical/observability/tree/main/blueprints/snaps).

### Prerequisites

- `snapcraft`
- `just`
- `yq`
- `gh` (GitHub CLI)

### Common Commands

```bash
# Build the snap locally
just pack

# Run tests
just test

# Update to latest upstream version
just update prometheus/statsd_exporter

# Fetch latest centralized files from canonical/observability
just refresh
```

## License

Apache License 2.0
