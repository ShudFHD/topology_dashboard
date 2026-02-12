# Kubernetes Workload Emulation & Topology Viewer

A comprehensive system for emulating Kubernetes workloads and visualizing ContainerLab topologies with real-time metrics monitoring and Liqo-style virtual pod management.

## Overview

This system provides:
- **Interactive Topology Visualization** - Web-based viewer for ContainerLab network topologies
- **Workload Emulation** - High-fidelity emulation of server workloads using KWOK fake pods
- **Metrics Collection** - Real-time CPU, memory, power, and PSI metrics
- **Virtual Pod Management** - Liqo-style cross-cluster pod offloading simulation
- **Monitoring Integration** - Prometheus & Grafana dashboards

## Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                     Host VM                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Flask Web Application (port 8080)                     │ │
│  │  - Topology Viewer UI                                   │ │
│  │  - Virtual Pod Manager                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Central Prometheus (port 9091)                        │ │
│  │  - Collects metrics from all nodes                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           ContainerLab Topology                       │  │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                 │  │
│  │  │ serf1  │  │ serf2  │  │ serf3  │  ...             │  │
│  │  │ K3s +  │  │ K3s +  │  │ K3s +  │                  │  │
│  │  │ KWOK   │  │ KWOK   │  │ KWOK   │                  │  │
│  │  └────────┘  └────────┘  └────────┘                  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Ubuntu 24.04 (or similar Linux distribution)
- Docker
- ContainerLab
- Python 3.8+
- Required Python packages:
```bash
  pip install flask pyyaml docker requests psutil
```

## Installation

### 1. Clone the Repository
```bash
git clone <your-repo-url>
cd <repo-directory>
```

### 2. Deploy ContainerLab Topology
```bash
sudo clab deploy -t topology.yaml
```

### 3. Setup Web Application
```bash
cd flask-app/
pip install -r requirements.txt
```

### 4. Configure Prometheus

Edit `prometheus.yml` to include all your serf nodes:
```yaml
scrape_configs:
  - job_name: 'kwok-emulation'
    static_configs:
      - targets:
          - 'clab-emulation-serf1:9090'
          - 'clab-emulation-serf2:9090'
          - 'clab-emulation-serf3:9090'
          # Add all your nodes...
```

Start Prometheus:
```bash
prometheus --config.file=prometheus.yml --web.listen-address=:9091
```

## Running the Emulation

### Step 1: Start Workload Emulation

Run this script to set up emulation on all nodes (adjust the loop count to match your number of serf nodes):
```bash
#!/bin/bash
# setup_emulation.sh

# Complete setup loop for all serf containers
for i in {1..11}; do
    echo "========================================="
    echo "Setting up serf${i}..."
    echo "========================================="
    
    # 1. Create KWOK resources (nodes, namespaces, pods)
    docker exec clab-emulation-serf${i} bash -c \
        "cd /opt/annotations && python3 create_resources.py --config emulation_config.json"
    
    # 2. Start replay script in background (replays metrics to pods)
    docker exec -d clab-emulation-serf${i} bash -c \
        "cd /opt/annotations && python3 replay_metrics.py --config emulation_config.json --interval 5 --loop --max-concurrent 3 --batch-size 20"
    
    # 3. Start metrics exporter in background (exposes Prometheus metrics)
    docker exec -d clab-emulation-serf${i} bash -c \
        "cd /opt/annotations && python3 expose_metrics.py --config emulation_config.json --port 9090 --update-interval 2"
    
    echo "✓ serf${i} setup complete"
    sleep 2
done

echo ""
echo "========================================="
echo "All nodes configured!"
echo "========================================="
echo "Waiting 30 seconds for services to start..."
sleep 30

# Verify all targets are up in Prometheus
echo "Checking Prometheus targets..."
curl -s http://localhost:9091/api/v1/targets | \
    jq '.data.activeTargets[] | select(.labels.job=="kwok-emulation") | {instance: .labels.instance, health: .health}'
```

Make it executable and run:
```bash
chmod +x setup_emulation.sh
./setup_emulation.sh
```

### Step 2: Start the Web Application
```bash
cd flask-app/
python3 app.py
```

Access the UI at: `http://localhost:8080`

## Features

### 1. Topology Visualization

- **Interactive Network Graph**: Visualize your ContainerLab topology with vis.js
- **Node Information**: Click nodes to see cluster details, pods, and metrics
- **Real-time Status**: Color-coded nodes based on CPU load (green/amber/red)

### 2. Metrics Monitoring

- **Per-Pod Metrics**: View CPU, Memory, PSI, and Power consumption for individual pods
- **Node Aggregation**: See aggregated metrics at the node level
- **Time Range Selection**: 5min, 15min, 1h, 2h windows
- **Auto-refresh**: Live updating graphs (5-second intervals)
- **Comparison**: Side-by-side view of emulated vs real metrics

### 3. Liqo Virtual Connections

#### Create Liqo Connection
1. Right-click on a K3s node (serf node)
2. Select **"Liqo Connect"**
3. Click on destination K3s node
4. Connection established (green dashed line appears)

#### Create Virtual Pod
1. Right-click on source node
2. Select **"Create Pod"**
3. Click on destination node (must have Liqo connection)
4. Select workload template
5. Watch pod transfer animation

#### Manage Virtual Pods
- View all virtual pods in the sidebar
- See pod details (source/destination, workload)
- Delete virtual pods when no longer needed

#### Remove Liqo Connections
1. Right-click on node
2. Select **"Remove Liqo Links"**
3. Select connections to remove
4. System automatically deletes associated virtual pods

### 4. Workload Templates

Templates define time-series metrics profiles. Located in `workload_templates/`:
```json
{
  "time_series": [
    {"cpu": 500, "memory": 256, "power": 12.5, "psi": 5.2},
    {"cpu": 600, "memory": 280, "power": 14.0, "psi": 6.1}
  ]
}
```

## Configuration

### Flask Application (`config.yaml`)
```yaml
containerlab:
  topology_file: "../topology.yaml"

monitoring:
  central_prometheus:
    host: "localhost"
    port: 9091
  vm_prometheus_port: 9091

server:
  host: "0.0.0.0"
  port: 8080
  debug: true
```

### Emulation Config (`emulation_config.json`)

Located in each container at `/opt/annotations/emulation_config.json`:
```json
{
  "metadata": {
    "total_pods": 80,
    "total_namespaces": 4,
    "time_points": 132
  },
  "node_config": {
    "mode": "single",
    "single_node": {
      "name": "emulation-node-1",
      "cpu": "16",
      "memory": "60Gi"
    }
  },
  "namespaces": ["ns1", "ns2", "ns3", "ns4"],
  "pods": [...]
}
```

## Troubleshooting

### Emulation not starting

Check if resources were created:
```bash
docker exec clab-emulation-serf1 k3s kubectl get nodes
docker exec clab-emulation-serf1 k3s kubectl get pods --all-namespaces
```

### Metrics not appearing in Prometheus

Check if exporters are running:
```bash
docker exec clab-emulation-serf1 ps aux | grep expose_metrics
curl http://localhost:9090/metrics  # From inside container
```

### Web UI not loading topology

Check Flask logs for errors:
```bash
# In the Flask terminal
# Look for error messages
```

Verify ContainerLab is running:
```bash
sudo clab inspect
```

### Virtual pods not appearing

Check registry file:
```bash
cat flask-app/virtual_pods/registry.json
```

Check Liqo connections:
```bash
cat flask-app/virtual_pods/liqo_connections.json
```

## Clean Up

### Stop emulation on a specific node
```bash
docker exec clab-emulation-serf1 bash -c \
    "pkill -f replay_metrics.py && pkill -f expose_metrics.py"
```

### Delete KWOK resources
```bash
docker exec clab-emulation-serf1 bash -c \
    "cd /opt/annotations && python3 create_resources.py --config emulation_config.json --delete"
```

### Stop all and destroy topology
```bash
# Stop Flask app (Ctrl+C)
# Stop Prometheus (Ctrl+C)

# Destroy ContainerLab topology
sudo clab destroy -t topology.yaml
```

## Project Structure
```
.
├── flask-app/
│   ├── app.py                      # Main Flask application
│   ├── config.yaml                 # Application configuration
│   ├── static/
│   │   ├── css/
│   │   │   └── style.css           # UI styling
│   │   ├── js/
│   │   │   └── topology.js         # Frontend logic
│   │   ├── images/                 # Node icons
│   │   └── index.html              # Main UI
│   ├── scripts/
│   │   └── virtual_pod_manager.py  # Virtual pod CLI tool
│   ├── workload_templates/         # Workload JSON files
│   └── virtual_pods/
│       ├── registry.json           # Virtual pods state
│       └── liqo_connections.json   # Liqo connections state
├── topology.yaml                    # ContainerLab topology
└── prometheus.yml                   # Prometheus configuration
```

## API Endpoints

- `GET /api/topology` - Get topology structure
- `GET /api/node/<node>/cluster-info` - Get K3s cluster details
- `GET /api/node/<node>/timeseries` - Get node metrics
- `GET /api/node/<node>/pod/<ns>/<pod>/timeseries` - Get pod metrics
- `GET /api/node/<node>/emulation-config` - Get emulation resources
- `GET /api/virtual-pods` - List virtual pods
- `POST /api/virtual-pods/create` - Create virtual pod
- `DELETE /api/virtual-pods/<id>` - Delete virtual pod
- `GET /api/liqo-connections` - List Liqo connections
- `POST /api/liqo-connections` - Create Liqo connection
- `DELETE /api/liqo-connections` - Remove Liqo connection
- `GET /api/workload-templates` - List available workload templates

## Credits

Developed for Kubernetes workload emulation research at [Your Institution].

## License

[Your License]
