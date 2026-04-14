# InfluxDB Enterprise Data Node Reindexing in Kubernetes

When a data node in InfluxDB Enterprise needs to be reindexed, the process requires `influxd` to not be running. This guide covers how to access the data files in Kubernetes without the InfluxDB pod running.

## Prerequisites

- Access to the cluster via `kubectl`
- `influx_inspect` tool available in the InfluxDB image
- PVC names for your data nodes (retrieve with `kubectl get pvc -n <namespace>`)

## Step 1: Scale Down the Data Node StatefulSet

```bash
kubectl scale statefulset <data-statefulset-name> --replicas=0 -n <namespace>
```

This terminates the data node pods but preserves the PVCs.

## Step 2: Identify PVC Names

```bash
kubectl get pvc -n <namespace>
```

For InfluxDB Enterprise, data PVCs follow a pattern like:
- `influxdb-enterprise-v1-data-data-influxdb-enterprise-v1-data-0`
- `influxdb-enterprise-v1-data-data-influxdb-enterprise-v1-data-1`

## Step 3: Create a Maintenance Pod

Create a YAML file for the maintenance pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: influx-maintenance-0
spec:
  securityContext:
    runAsUser: 0
    fsGroup: 999
  initContainers:
  - name: fix-permissions
    image: influxdb:<version>
    command: ["/bin/sh", "-c", "chown -R 999:999 /data"]
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: maintenance
    image: influxdb:<version>
    securityContext:
      runAsUser: 999
    command: ["/bin/sh", "-c", "echo y | influx_inspect buildtsi -datadir /data/data -waldir /data/wal"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <pvc-name>
```

Apply it:
```bash
kubectl apply -f maintenance-pod.yaml -n <namespace>
```

## Step 4: Run the Reindex

Watch the logs:
```bash
kubectl logs influx-maintenance-0 -n <namespace> -f
```

The reindex will skip shards that already have valid TSI indexes.

## Step 5: Clean Up

When complete:
```bash
kubectl delete pod influx-maintenance-0 -n <namespace>
```

## Step 6: Scale Up the StatefulSet

```bash
kubectl scale statefulset <data-statefulset-name> --replicas=<original-count> -n <namespace>
```

## Repeat for Additional Nodes

For each additional data node:
1. Create a new maintenance pod YAML with the corresponding PVC name
2. Apply and wait for completion
3. Delete the maintenance pod

## Notes

- The `initContainer` fixes ownership to ensure the main container (running as UID 999) can write to the data directories.
- The `echo y` handles the prompt from `influx_inspect` warning about running as root.
- Using a StatefulSet with stable PVCs ensures the same volume is reattached when scaling back up
