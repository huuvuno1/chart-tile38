/!\ Unmaintained → **Now Production-Ready! ✨**

# Tile38 Helm Chart (Production-Ready)

This is a production-ready Helm chart to install [Tile38](http://tile38.com) on a Kubernetes cluster with full leader-follower replication support.

## Features

- ✅ **Leader-Follower Replication** - Fully configured according to Tile38 official documentation
- ✅ **StatefulSet for Leader** - Stable network identity and persistent storage
- ✅ **Health Checks** - Liveness and readiness probes with `/healthz` endpoint
- ✅ **Production Resources** - Sensible CPU/Memory limits and requests
- ✅ **High Availability** - PodDisruptionBudgets for safer cluster operations
- ✅ **Caught-Up Validation** - Follower readiness checks ensure replication is synced
- ✅ **Persistent Storage** - Separate PVCs for leader and followers

## TL;DR;

```bash
$ git clone https://github.com/nouney/chart-tile38
$ helm install my-tile38 ./chart-tile38
```

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure

## Architecture

The chart deploys:
- **1 Leader StatefulSet**: Accepts all write operations (SET, DEL, etc.)
- **N Follower Deployments**: Read-only replicas that follow the leader
- **Automatic Replication**: Followers automatically connect to leader via `FOLLOW` command
- **Health Monitoring**: Followers only accept reads when caught up with leader

## Configuration

The following table lists the configurable parameters of the Tile38 chart and their default values.

| Parameter                  | Description                                     | Default                                                    |
| -----------------------    | ---------------------------------------------   | ---------------------------------------------------------- |
| `image.repository`         | `tile38` image repository                       | `tile38/tile38`                                            |
| `image.tag`                | `tile38` image tag                              | `1.33.3`                                                   |
| `image.pullPolicy`         | Image pull policy                               | `IfNotPresent`                                             |
| `service.port`             | TCP port                                        | `9851`                                                     |
| `service.type`             | k8s service type exposing ports, e.g. `NodePort`| `ClusterIP`                                                |
| `leader.replicaCount`      | Number of leader replicas                       | `1`                                                        |
| `leader.resources.requests.cpu`    | Leader CPU request                      | `250m`                                                     |
| `leader.resources.requests.memory` | Leader Memory request                   | `512Mi`                                                    |
| `leader.resources.limits.cpu`      | Leader CPU limit                        | `1000m`                                                    |
| `leader.resources.limits.memory`   | Leader Memory limit                     | `2Gi`                                                      |
| `leader.persistentVolume.enabled`      | Create a volume to store Tile38 data            | `true`                                         |
| `leader.persistentVolume.existingClaim`| Provide an existing PersistentVolumeClaim       | `nil`                                          |
| `leader.persistentVolume.storageClass` | Storage class of backing PVC                    | `nil` (uses alpha storage class annotation)    |
| `leader.persistentVolume.accessModes`  | Use volume as ReadOnly or ReadWrite             | `[ReadWriteOnce]`                              |
| `leader.persistentVolume.annotations`  | Persistent Volume annotations                   | `{}`                                           |
| `leader.persistentVolume.size`         | Size of data volume                             | `2Gi`                                          |
| `leader.persistentVolume.subPath`      | Subdirectory of the volume to mount at          | `""`                                           |
| `leader.persistentVolume.mountPath`    | Mount path of data volume                       | `/data`                                        |
| `leader.nodeSelector`             | Node labels for pod assignment                  | `{}`                                                       |
| `leader.affinity`                 | Affinity settings for pod assignment            | `{}`                                                       |
| `leader.tolerations`              | Toleration labels for pod assignment            | `[]`                                                       |
| `follower.replicaCount`    | Number of follower replicas                     | `1`                                                        |
| `follower.resources.requests.cpu`    | Follower CPU request                    | `250m`                                                     |
| `follower.resources.requests.memory` | Follower Memory request                 | `512Mi`                                                    |
| `follower.resources.limits.cpu`      | Follower CPU limit                      | `1000m`                                                    |
| `follower.resources.limits.memory`   | Follower Memory limit                   | `2Gi`                                                      |
| `follower.persistentVolume.enabled`      | Create a volume to store Tile38 data            | `true`                                     |
| `follower.persistentVolume.existingClaim`| Provide an existing PersistentVolumeClaim       | `nil`                                          |
| `follower.persistentVolume.storageClass` | Storage class of backing PVC                    | `nil` (uses alpha storage class annotation)    |
| `follower.persistentVolume.accessModes`  | Use volume as ReadOnly or ReadWrite             | `[ReadWriteOnce]`                              |
| `follower.persistentVolume.annotations`  | Persistent Volume annotations                   | `{}`                                           |
| `follower.persistentVolume.size`         | Size of data volume                             | `2Gi`                                          |
| `follower.persistentVolume.subPath`      | Subdirectory of the volume to mount at          | `""`                                           |
| `follower.persistentVolume.mountPath`    | Mount path of data volume                       | `/data`                                        |
| `follower.resources`                | CPU/Memory resource requests/limits             | See above                                                  |                                                         |
| `follower.nodeSelector`             | Node labels for pod assignment                  | `{}`                                                       |
| `follower.affinity`                 | Affinity settings for pod assignment            | `{}`                                                       |
| `follower.tolerations`              | Toleration labels for pod assignment            | `[]`                                                       |

## Production Deployment Guide

### Quick Start for Production

```bash
# Install with 1 leader and 2 followers for HA
helm install my-tile38 ./chart-tile38 \
  --set leader.replicaCount=1 \
  --set follower.replicaCount=2 \
  --set leader.resources.requests.cpu=500m \
  --set leader.resources.requests.memory=1Gi
```

### Accessing Tile38

**Write Operations (Leader only):**
```bash
kubectl port-forward svc/my-tile38-leader 9851:9851
# Then connect: redis-cli -p 9851
# SET fleet truck1 POINT 33.5123 -112.2693
```

**Read Operations (Follower - load balanced):**
```bash
kubectl port-forward svc/my-tile38-follower 9851:9851
# GET fleet truck1
# SCAN fleet
```

### Monitoring Replication

Check follower status:
```bash
kubectl exec -it <follower-pod> -- tile38-cli
> HEALTHZ
```

Expected output when caught up:
```json
{"ok":true,"following":{"leader":"my-tile38-leader:9851","caught_up":true}}
```

### Scaling

**Scale followers (recommended for read-heavy workloads):**
```bash
helm upgrade my-tile38 ./chart-tile38 --set follower.replicaCount=3
```

**Note:** Tile38 uses single-leader replication. Multiple leader replicas require external HA solution.

### Backup & Restore

```bash
# Backup from leader
kubectl exec <leader-pod> -- tile38-cli -raw AOF > backup.aof

# Restore to new deployment
kubectl cp backup.aof <new-leader-pod>:/data/appendonly.aof
kubectl delete pod <new-leader-pod>  # Restart to load data
```


### Existing PersistentVolumeClaims

1. Create the PersistentVolume
1. Create the PersistentVolumeClaim
1. Install the chart
```bash
$ helm install --set leader.persistentVolume.existingClaim=PVC_NAME chart-tile38
```
