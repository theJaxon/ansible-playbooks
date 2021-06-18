Prometheus 
=========

- Deploying prometheus using helm chart.
- The procedure follows Elton Stoneman's [ECS-V1: Monitoring with Prometheus and Grafana](https://youtu.be/JlescH2xFok)

Requirements
------------

- pip `Openshift` module should be installed as kubernetes and helm modules rely on it.

---

### Prometheus:
- `<prometheus-server>/config` contains the rules as provided in `prometheus.yml` ConfigMap.
- `<prometheus-server>/targets` shows the list of containers set as prometheus targets (Also from ConfigMap)

### Prometheus Metrics
```bash
# Get CPU Usage for containers
process_cpu_seconds_total (For node and go) | process_cpu_usage (For Java).

# Number of requests coming into the app 
http_server_requests_seconds_count

# Amount of times api was called 
access_log_total | iotd_api_image_load_total
```


