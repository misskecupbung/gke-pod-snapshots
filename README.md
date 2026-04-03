# GKE Pod Snapshots

Pod Snapshots let you save the memory state of a running container and restore it later. The pod starts from that saved state instead of running initialization again from the beginning.

GKE uses CRIU (Checkpoint/Restore In Userspace) to capture the memory image of a running container.

## What this covers

- Enabling Pod Snapshots on a GKE cluster
- Creating a PodSnapshot from a running pod
- Restoring a pod from a snapshot
- Measuring startup time difference with and without snapshots

## Prerequisites

- `gcloud` CLI authenticated
- `kubectl` installed
- GCP project with billing enabled
- APIs enabled:

```bash
gcloud services enable container.googleapis.com storage.googleapis.com
```

## Set variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_NAME=lab-pod-snapshots
export ZONE=us-central1-a
```

## Clone the repo

```bash
git clone https://github.com/misskecupbung/gke-pod-snapshots.git
cd gke-pod-snapshots
```

## Step 1 — Create a cluster with Pod Snapshots enabled

Pod Snapshots need the rapid release channel and a gVisor node pool. Create the cluster, then add the gVisor pool:

```bash
gcloud beta container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 1 \
  --release-channel rapid \
  --workload-pool=$PROJECT_ID.svc.id.goog \
  --disk-size=50 \
  --disk-type=pd-standard \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --enable-pod-snapshots
```

```bash
gcloud beta container node-pools create gvisor-pool \
  --cluster=$CLUSTER_NAME \
  --zone=$ZONE \
  --num-nodes=1 \
  --machine-type=n2-standard-4 \
  --disk-size=50 \
  --disk-type=pd-standard \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --sandbox type=gvisor
```

Pods must run in GKE Sandbox because Pod Snapshots depend on the gVisor runtime it provides. The `gvisor-pool` nodes have this. Get credentials and check the CRD is installed:

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
kubectl get crd | grep snapshot
```

You should see `podsnapshots.snapshot.gke.io`.

## Step 2 — Deploy the slow-start app

Before using snapshots, we need a baseline. This app waits 30 seconds on startup to simulate model loading, then serves HTTP.

```bash
kubectl apply -f manifests/slow-app.yaml
kubectl get pods -w
```

Wait until the pod is `Running`. It takes about 35 seconds. Check it's working:

```bash
POD=$(kubectl get pod -l app=slow-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- python3 -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8080/ready').read().decode())" from pod events. We'll compare this number after the restore in Step 4:

```bash
kubectl describe pod $POD | grep -A5 "Started\|Ready"
```

## Step 3 — Create a snapshot

Now take the snapshot while the app is running. The container stays up during this process.

```bash
kubectl apply -f manifests/pod-snapshot.yaml
kubectl get podsnapshot -w
```

Wait for the status to show `Completed`. This takes 30–60 seconds. GKE freezes the container briefly, saves the memory image to GCS, then unfreezes it.

```bash
kubectl describe podsnapshot slow-app-snapshot
```

The `Status.Checkpoint.Uri` field shows the GCS path where the snapshot is stored.

## Step 4 — Delete the pod and restore from snapshot

Delete the original pod and start a new one from the snapshot. This is where you see the difference.

```bash
kubectl delete -f manifests/slow-app.yaml
kubectl apply -f manifests/slow-app-restore.yaml
kubectl get pods -w
```

The pod reaches `Running` in a few seconds. The 30-second init did not run — the process loaded from the saved state.

```bash
POD=$(kubectl get pod -l app=slow-app-restore -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- python3 -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8080/ready').read().decode())"
```

`{"status":"ready","initialized":true}` — already initialized.

Compare this startup time with what you recorded in Step 2.

## Step 5 — Use with a Deployment

In Step 4 we restored a single pod manually. In practice, you want every restart to use the snapshot automatically. Add the annotation to the Deployment template and GKE will load from the snapshot on each pod restart — crash, rolling update, or node drain.

```bash
kubectl apply -f manifests/slow-app-deployment.yaml
```

Delete a pod to trigger a replacement and see how fast it comes up:

```bash
kubectl delete pod -l app=slow-app-deployment --wait=false
kubectl get pods -l app=slow-app-deployment -w
```

## Step 6 — Clean up

```bash
kubectl delete -f manifests/
kubectl delete podsnapshot --all
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

## Limitations

Not all apps work well with snapshots.

- **GKE Sandbox required.** Pods must run in GKE Sandbox (gVisor). That's why the lab uses a separate `gvisor-pool` node pool.
- **No E2 machine types.** Pod Snapshots don't support E2 machines. Use N2 or other supported families.
- **No Cloud Storage FUSE CSI sidecar.** If your pod uses the GCS Fuse CSI driver sidecar, snapshots won't work.
- **Network connections.** TCP connections in the snapshot are no longer valid on restore. Apps that open new connections when needed are fine. Apps that keep long-lived connections open may fail.
- **Expired credentials.** Tokens, leases, and sessions may have expired by restore time. The app needs to handle re-authentication.
- **Node-specific state.** If init stored something tied to the original node's IP or identity, restoring on a different node can break things.
- **Snapshot freshness.** If the content changes — new model weights, updated config — the old snapshot is no longer useful. Create a new one. `maxSnapshots: 3` keeps the last 3 versions.

The best fit is a stateless server that loads a large file or model once, then handles requests.

This feature is in Preview. Check the [GKE docs](https://cloud.google.com/kubernetes-engine/docs/concepts/pod-snapshots) for current status.
