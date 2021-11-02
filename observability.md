# Observability: Telemetry and Logs

## Overview

In this section, we will learn about a couple of monitoring (Prometheus), tracing (Zipkin), and data visualization tools (Grafana).

For Grafana and Kiali to work, we will first have to install the Prometheus addon.

## Observability and Prometheus 

Thanks to the sidecar deployment model where Envoy proxies run next to application instances and intercpt traffic, these proxies collect metrics.

The metrics Envoy proxies collect and helping us get visibility into the state of your system. Gaining this visibility into our systems is critical because we need to understand what's happening and empower the operators to troubleshoot, maintain, and optimize applications. 

Istio generates three types of telemetry to provide observability to services in the mesh:
- Metrics
- Distributed Traces
- Access Logs

### Metrics

Istio generates metrics based on the four golden signals: latency, traffic, errors and saturation.

<strong>Latency</strong> represents the time it takes to service a request. These metrics should be broken down into latency of succesful requests (e.g. HTTP 200) and failed requests (e.g. HTTP 500).