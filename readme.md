# Deploy Perses on a Kind Cluster — Step by Step

## Prerequisites

- `docker` installed and running
- `kind` CLI installed
- `kubectl` CLI installed
- `helm` CLI installed

---

## Step 1: Create a Kind Cluster

```bash
kind create cluster --name perses-lab
```

Verify:

```bash
kubectl cluster-info --context kind-perses-lab
```

---

## Step 2: Deploy Prometheus (the datasource backend)

Perses needs a Prometheus instance to query. Install it via the community Helm chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set server.service.type=ClusterIP \
  --set alertmanager.enabled=false \
  --set kube-state-metrics.enabled=true \
  --set prometheus-node-exporter.enabled=true \
  --set prometheus-pushgateway.enabled=false
```

Wait for it to come up:

```bash
kubectl -n monitoring rollout status deploy/prometheus-server
```

The in-cluster URL will be: `http://prometheus-server.monitoring.svc.cluster.local:80`

---

## Step 3: Deploy Perses via Helm

```bash
helm repo add perses https://perses.github.io/helm-charts
helm repo update
```

Create a `perses-values.yaml` file:

```yaml
image:
  registry: docker.io
  name: persesdev/perses
  version: "v0.53.1"

replicas: 1

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

config:
  security:
    readonly: false
    enable_auth: false
  database:
    file:
      folder: /perses
      extension: json
```

> **Note:** The Helm chart has a `datasources` field for provisioning datasources at
> install time, but it is deprecated and does not work — the provisioning volume is
> never mounted, so the files are not picked up. We will create the datasource via
> the API in Step 6 instead.

### Known issue: Helm file size limit

Helm v3.17.3+ introduced a 5 MB per-file limit that breaks the Perses chart
installation. Running `helm install perses perses/perses ...` directly will fail
with:

```
Error: INSTALLATION FAILED: chart file "pack-*.pack" is larger than the maximum file size 5242880
```

**Workaround:** Pull the chart first, then install from the local directory:

```bash
helm pull perses/perses --untar --untardir /tmp/perses-chart

helm install perses /tmp/perses-chart/perses \
  --namespace perses --create-namespace \
  -f perses-values.yaml
```

Wait for it:

```bash
kubectl -n perses rollout status statefulset/perses --timeout=120s
```

> Perses deploys as a **StatefulSet** (not a Deployment) when file-based storage is
> enabled, so use `statefulset/perses` in rollout commands.

---

## Step 4: Access the Perses UI

Port-forward the Perses service to your local machine:

```bash
kubectl -n perses port-forward svc/perses 8080:8080 &
```

Open your browser at **http://localhost:8080**. You should see the Perses UI.

---

## Step 5: Verify Prometheus is reachable from Perses

Before creating the datasource, confirm that the Perses pod can reach Prometheus:

```bash
kubectl -n perses exec statefulset/perses -- \
  wget -qO- "http://prometheus-server.monitoring.svc.cluster.local:80/api/v1/query?query=up" \
  | head -c 200
```

You should see a JSON response with `"status":"success"`.

---

## Step 6: Create a Global Datasource

Create the Prometheus datasource via the Perses REST API.

> **Important:** Use `proxy` mode, not `directUrl`. With `directUrl`, the *browser*
> tries to call the Prometheus URL directly — which is an in-cluster address
> unreachable from your laptop, resulting in a **"Failed to fetch"** error. With
> `proxy` mode, queries are routed through the Perses server, which *can* reach the
> in-cluster URL.

```bash
curl -s -X POST http://localhost:8080/api/v1/globaldatasources \
  -H 'Content-Type: application/json' \
  -d '{
    "kind": "GlobalDatasource",
    "metadata": {
      "name": "prometheus"
    },
    "spec": {
      "default": true,
      "plugin": {
        "kind": "PrometheusDatasource",
        "spec": {
          "proxy": {
            "kind": "HTTPProxy",
            "spec": {
              "url": "http://prometheus-server.monitoring.svc.cluster.local:80"
            }
          }
        }
      }
    }
  }' | python3 -m json.tool
```

Verify the proxy path works:

```bash
curl -s "http://localhost:8080/proxy/globaldatasources/prometheus/api/v1/query?query=up" \
  | python3 -m json.tool | head -20
```

In the UI, navigate to **Admin > Global Datasources** — the "prometheus" entry
should appear.

---

## Step 7: Create a Project

Dashboards live inside projects. Create one:

```bash
curl -s -X POST http://localhost:8080/api/v1/projects \
  -H 'Content-Type: application/json' \
  -d '{
    "kind": "Project",
    "metadata": {
      "name": "my-project"
    }
  }' | python3 -m json.tool
```

---

## Step 8: Create a Dashboard

Create a dashboard with three panels — a time-series chart, a stat, and a gauge:

```bash
curl -s -X POST http://localhost:8080/api/v1/projects/my-project/dashboards \
  -H 'Content-Type: application/json' \
  -d '{
  "kind": "Dashboard",
  "metadata": {
    "name": "node-overview",
    "project": "my-project"
  },
  "spec": {
    "display": {
      "name": "Node Overview"
    },
    "duration": "1h",
    "refreshInterval": "30s",
    "variables": [
      {
        "kind": "ListVariable",
        "spec": {
          "name": "instance",
          "allowMultiple": false,
          "allowAllValue": true,
          "plugin": {
            "kind": "PrometheusLabelValuesVariable",
            "spec": {
              "labelName": "instance",
              "matchers": ["up"]
            }
          }
        }
      }
    ],
    "panels": {
      "cpuUsage": {
        "kind": "Panel",
        "spec": {
          "display": {
            "name": "CPU Usage Over Time"
          },
          "plugin": {
            "kind": "TimeSeriesChart",
            "spec": {
              "legend": {
                "position": "bottom"
              }
            }
          },
          "queries": [
            {
              "kind": "TimeSeriesQuery",
              "spec": {
                "plugin": {
                  "kind": "PrometheusTimeSeriesQuery",
                  "spec": {
                    "query": "1 - avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\", instance=~\"$instance\"}[5m]))",
                    "seriesNameFormat": "{{instance}}"
                  }
                }
              }
            }
          ]
        }
      },
      "memoryUsed": {
        "kind": "Panel",
        "spec": {
          "display": {
            "name": "Memory Used %"
          },
          "plugin": {
            "kind": "StatChart",
            "spec": {
              "calculation": "last-number",
              "format": {
                "unit": "percent"
              }
            }
          },
          "queries": [
            {
              "kind": "TimeSeriesQuery",
              "spec": {
                "plugin": {
                  "kind": "PrometheusTimeSeriesQuery",
                  "spec": {
                    "query": "100 - ((node_memory_MemAvailable_bytes{instance=~\"$instance\"} * 100) / node_memory_MemTotal_bytes{instance=~\"$instance\"})"
                  }
                }
              }
            }
          ]
        }
      },
      "targetsUp": {
        "kind": "Panel",
        "spec": {
          "display": {
            "name": "Targets Up"
          },
          "plugin": {
            "kind": "GaugeChart",
            "spec": {
              "calculation": "last-number",
              "format": {
                "unit": "decimal"
              },
              "thresholds": {
                "steps": [
                  {"value": 1},
                  {"value": 3}
                ]
              }
            }
          },
          "queries": [
            {
              "kind": "TimeSeriesQuery",
              "spec": {
                "plugin": {
                  "kind": "PrometheusTimeSeriesQuery",
                  "spec": {
                    "query": "count(up{instance=~\"$instance\"} == 1)"
                  }
                }
              }
            }
          ]
        }
      }
    },
    "layouts": [
      {
        "kind": "Grid",
        "spec": {
          "display": {
            "title": "Overview",
            "collapse": {
              "open": true
            }
          },
          "items": [
            {
              "x": 0, "y": 0,
              "width": 12, "height": 4,
              "content": { "$ref": "#/spec/panels/memoryUsed" }
            },
            {
              "x": 12, "y": 0,
              "width": 12, "height": 4,
              "content": { "$ref": "#/spec/panels/targetsUp" }
            },
            {
              "x": 0, "y": 4,
              "width": 24, "height": 8,
              "content": { "$ref": "#/spec/panels/cpuUsage" }
            }
          ]
        }
      }
    ]
  }
}' | python3 -m json.tool
```

> **Note on `$ref` paths:** Always use `#/spec/panels/<panelName>`, not
> `#/spec/config/panels/<panelName>`. Perses resolves `$ref` relative to the
> dashboard config root.

---

## Step 9: View the Dashboard

In your browser at **http://localhost:8080**, navigate to:

**Projects > my-project > Dashboards > Node Overview**

You should see:
- A **stat panel** showing memory usage percentage
- A **gauge panel** showing the count of healthy scrape targets
- A **time-series chart** showing CPU usage over time
- An **instance variable** dropdown at the top for filtering

---

## Cleanup

```bash
# Stop the port-forward
kill %1

# Delete the Kind cluster
kind delete cluster --name perses-lab

# Remove the local chart
rm -rf /tmp/perses-chart
```

---

## Quick Reference

| Resource | API Endpoint |
|---|---|
| Global Datasources | `GET/POST /api/v1/globaldatasources` |
| Projects | `GET/POST /api/v1/projects` |
| Dashboards | `GET/POST /api/v1/projects/{project}/dashboards` |
| Project Datasources | `GET/POST /api/v1/projects/{project}/datasources` |
| Proxy (global) | `GET /proxy/globaldatasources/{name}/...` |
| Proxy (project) | `GET /proxy/projects/{project}/datasources/{name}/...` |

---

## Troubleshooting

### Helm install fails with "larger than the maximum file size 5242880"

Helm v3.17.3+ introduced a hard 5 MB per-file limit. Pull the chart locally first,
then install from the directory. See Step 3.

### GlobalDatasource not showing in the UI after Helm install

The Helm chart's `datasources` field is deprecated and broken — the provisioning
volume (`/etc/perses/provisioning`) is never created. Create datasources via the
REST API instead. See Step 6.

### "Failed to fetch" when browsing dashboards or datasources

The datasource is configured with `directUrl`, which makes the **browser** call the
Prometheus URL directly. In-cluster URLs like
`http://prometheus-server.monitoring.svc.cluster.local` are not reachable from your
laptop. Switch to `proxy` mode so queries go through the Perses server. See Step 6.

---

Perses version: **v0.53.1** | Helm chart: **0.21.0**
