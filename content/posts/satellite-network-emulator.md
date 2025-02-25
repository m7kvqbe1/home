+++
title = 'Network Emulation (tc netem)'
date = 2025-02-25T12:00:00+00:00
draft = false
+++

Using Linux traffic control (tc) and netem, we can create realistic satellite link conditions from low-earth orbit (LEO) to geostationary (GEO) satellites.

The great thing about this approach is that it's completely portable - you can drop it in front of any service without modifying the target application.

## Overview

- **Realistic Satellite Emulation**: Simulate various satellite scenarios from LEO to GEO
- **Real-time Monitoring**: Expose metrics via Prometheus to visualize in Grafana
- **Runtime Control**: Change network conditions on the fly
- **Portable Proxy Design**: Drop-in containerized solution that works with any service

## Technical Implementation

Here's how to build a network emulator using common Linux tools and containers:

### Containerized Proxy Architecture

Implement as a self-contained Docker container that acts as a transparent proxy:

```yaml
services:
  # Your target application
  app:
    image: your-app:latest

  # Network emulator proxy
  emulator:
    image: network-emulator:latest
    environment:
      - UPSTREAM_HOST=app
      - MODE=cycle
      - CYCLE_SCENARIOS=low_latency,high_latency
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - app

  # Optional monitoring
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus:/etc/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - ./monitoring/grafana:/var/lib/grafana
    ports:
      - "3000:3000"
```

This containerized approach provides several benefits:

1. **Portable**: Run it locally, in CI/CD, or in production
2. **Isolated**: Network conditions don't affect the host system
3. **Self-contained**: Includes all necessary tools and monitoring

### Network Proxy with socat

At the heart of the emulator is `socat`, a versatile networking relay tool. It acts as a transparent proxy, forwarding traffic between the client and backend server while allowing us to apply network conditions:

```bash
# TCP forwarding with IPv4/IPv6 support
socat -v TCP-LISTEN:80,fork,reuseaddr TCP:${UPSTREAM_HOST}:80
```

### Network Emulation with tc netem

The network conditions are applied using Linux's traffic control (`tc`) with the `netem` module. Here's how different satellite scenarios are implemented:

```bash
# LEO satellite with good conditions
tc qdisc add dev eth0 root netem \
  delay 400ms 30ms \
  loss 0.5% \
  rate 9mbit

# GEO satellite during heavy rain
tc qdisc add dev eth0 root netem \
  delay 600ms 100ms \
  loss 10% \
  corrupt 2% \
  rate 2mbit
```

We can dynamically update the conditions using a control script to monitor a named pipe for commands during runtime:

```bash
# Switch to heavy rain scenario
echo "set heavy_rain_satellite" > /tmp/netem_control

# Start automatic cycling
echo "cycle" > /tmp/netem_control

# Remove all network conditions
echo "set none" > /tmp/netem_control
```

```bash
# Create control pipe
mkfifo /tmp/netem_control

# Monitor for commands
while true; do
  if read command < /tmp/netem_control; then
    case "$command" in
      "set "*)
        scenario="${command#set }"
        apply_scenario "$scenario"
        ;;
      "cycle")
        start_cycle
        ;;
    esac
  fi
done
```

### Metrics Collection

Handle metrics collection via a custom exporter that provides Prometheus-compatible metrics, see trivial Golang example below:

```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
    "os/exec"
    "regexp"
    "strconv"
    "time"
)

var (
    networkDelay = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "network_delay_ms",
        Help: "Current network delay in milliseconds",
    })

    packetLoss = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "packet_loss_percent",
        Help: "Current packet loss percentage",
    })

    bandwidth = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "bandwidth_kbps",
        Help: "Current bandwidth in Kbps",
    })
)

func init() {
    prometheus.MustRegister(networkDelay)
    prometheus.MustRegister(packetLoss)
    prometheus.MustRegister(bandwidth)
}

func collectMetrics() {
    for {
        cmd := exec.Command("tc", "-s", "qdisc", "show", "dev", "eth0")
        output, err := cmd.Output()
        if err != nil {
            continue
        }

        // Parse tc output using regexp
        if delay := parseDelay(string(output)); delay != nil {
            networkDelay.Set(*delay)
        }
        if loss := parseLoss(string(output)); loss != nil {
            packetLoss.Set(*loss)
        }
        if bw := parseBandwidth(string(output)); bw != nil {
            bandwidth.Set(*bw)
        }

        time.Sleep(1 * time.Second)
    }
}

func main() {
    go collectMetrics()

    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":9091", nil)
}
```

The metrics are exposed on `/metrics` in the standard Prometheus format:

```text
# HELP network_delay_ms Current network delay in milliseconds
# TYPE network_delay_ms gauge
network_delay_ms 600.0
# HELP packet_loss_percent Current packet loss percentage
# TYPE packet_loss_percent gauge
packet_loss_percent 1.0
# HELP bandwidth_kbps Current bandwidth in Kbps
# TYPE bandwidth_kbps gauge
bandwidth_kbps 2048.0
```

These metrics are then scraped by Prometheus and visualized in Grafana, providing real-time insights into the network conditions:

```yaml
scrape_configs:
  - job_name: "satellite-emulator"
    static_configs:
      - targets: ["localhost:9091"]
    scrape_interval: 1s
```

## Use Cases

This pattern can be useful for:

- Testing application behavior under various network conditions
- Evaluating protocol performance
- Network simulation and planning
- Automated testing in CI/CD pipelines
