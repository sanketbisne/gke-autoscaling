# GKE Autoscaling Demo Project

This project demonstrates Kubernetes autoscaling capabilities on Google Kubernetes Engine (GKE), including Horizontal Pod Autoscaler (HPA), Cluster Autoscaler, and rolling updates/rollbacks.

## 1. Cluster Creation

First, create the GKE cluster with the specified requirements:

```bash
# Set your project (replace with your actual project ID)
# gcloud config set project YOUR_PROJECT_ID

# Create the cluster
gcloud container clusters create autoscale-demo-cluster \
  --region asia-south1 \
  --num-nodes 1 \
  --machine-type e2-standard-4 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 2 \
  --node-pool-name general-pool

# Get cluster credentials
gcloud container clusters get-credentials autoscale-demo-cluster --region asia-south1
```

## 2. Architecture Explanation

### HPA vs Cluster Autoscaler
- **Horizontal Pod Autoscaler (HPA)** watches metrics (like CPU or Memory) of your Pods. When the metrics cross a defined threshold, it increases or decreases the number of Pod replicas in your Deployment.
- **Cluster Autoscaler** watches for Pods that cannot be scheduled because there are not enough resources (CPU/Memory) on the existing nodes in the cluster. When it sees these "Pending" Pods, it requests the cloud provider (GCP) to provision a new node. It will also remove nodes if they are underutilized.

### How Pending Pods Trigger Node Creation
When HPA scales up the number of replicas, the Kubernetes Scheduler tries to assign them to nodes. Because our application has resource `requests` (100m CPU), the node must have that much unallocated capacity. Once the single node runs out of capacity, new Pods stay in a `Pending` state. The Cluster Autoscaler detects these pending pods and automatically provisions a new `e2-standard-4` node.

### Rolling Updates Internally
When you update a Deployment (e.g., changing the image version), Kubernetes doesn't kill all old Pods at once. It starts a new ReplicaSet and scales it up slightly, waits for the new Pods to pass their Readiness probes, and then scales down the old ReplicaSet. This ensures zero downtime because traffic is only sent to healthy Pods.

### Why Probes are Important
- **Liveness Probe**: Tells Kubernetes if the container is dead or hung. If it fails, Kubernetes restarts the container.
- **Readiness Probe**: Tells Kubernetes if the container is ready to accept traffic. If it fails, Kubernetes stops sending traffic to the Pod. This is crucial during rolling updates so users don't hit a starting-up container.

## 3. Production Best Practices Demonstrated
1. **Resource Requests/Limits**: Essential for predictable scheduling and preventing noisy neighbor problems. Limits prevent a single container from consuming all node resources.
2. **Avoid Latest Image Tag**: We use `nginx:1.24.0-alpine` instead of `latest` for deterministic deployments and reliable rollbacks.
3. **Probes**: Configured for graceful traffic handling and auto-recovery.
4. **Least Privilege**: The load generator uses the minimal `busybox` image. (Ideally, runAsNonRoot and securityContexts would also be added for strict production environments).

---

## 4. Visual Demo Flow

Follow these steps while presenting to the class.

### Step 1: Deploy Application
Open two terminal windows. In the first window, apply the manifests:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl apply -f load-generator.yaml
```

In the second window, start watching the cluster state:
```bash
# Watch pods, hpa, and nodes in the autoscale-demo namespace
watch -n 1 'kubectl get hpa,pods -n autoscale-demo; echo ""; kubectl get nodes'
```

### Step 2: Enable Autoscaling (Generate Load)
Explain that the application is running, and the HPA is monitoring it. We will now generate load to trigger autoscaling.

```bash
# Scale the load generator to start hitting the service
kubectl scale deployment load-generator -n autoscale-demo --replicas=20
```

### Step 3: Observe Autoscaling
*Narration:* "As the load generator sends traffic, the CPU utilization of our sample-app will rise. Once it crosses 50%, HPA will start adding new replicas."

Watch the output in the second terminal:
- You will see the HPA `TARGETS` percentage increase.
- New `sample-app` Pods will be created.
- Eventually, the single node will run out of space, and Pods will enter a `Pending` state.
- *Narration:* "Notice the Pending pods. This tells the Cluster Autoscaler we need more capacity. It will now spin up a second node (Max 2)."
- Wait ~1-2 minutes for the new node to appear and the Pending pods to start running.
- **Tip:** Run `kubectl get pods -n autoscale-demo -o wide` and point out the `NODE` column to show students that the new pods are being scheduled onto the newly provisioned node!

### Step 4: Perform Rolling Update
Once the cluster has scaled out, demonstrate a zero-downtime deployment.

```bash
# Update the image to a newer version
kubectl set image deployment/sample-app app=nginx:1.25.4-alpine -n autoscale-demo

# Watch the rollout status
kubectl rollout status deployment/sample-app -n autoscale-demo
```
*Narration:* "Kubernetes is rolling out the new image. It creates new pods, waits for them to become 'Ready' using the readiness probe, and then terminates the old pods. The service continues routing traffic without interruption."

### Step 5: Perform Rollback
If the new version has issues, we can easily roll back.

```bash
# Check rollout history
kubectl rollout history deployment/sample-app -n autoscale-demo

# Undo the last rollout
kubectl rollout undo deployment/sample-app -n autoscale-demo

# Verify it's restoring
kubectl rollout status deployment/sample-app -n autoscale-demo
```

### Step 6: Cleanup
After the demo, scale down the load generator to watch the cluster scale back down (this takes ~5 minutes for HPA, and ~10 minutes for Cluster Autoscaler).

```bash
# Stop the load
kubectl scale deployment load-generator -n autoscale-demo --replicas=0
```
Or delete the entire namespace:
```bash
kubectl delete -f .
```

## Useful Monitoring Commands
- `kubectl get hpa -n autoscale-demo -w` (Watch HPA metrics dynamically)
- `kubectl get pods -n autoscale-demo -w` (Watch pod transitions)
- `kubectl top pods -n autoscale-demo` (View CPU/Memory usage per pod)
- `kubectl top nodes` (View CPU/Memory usage per node)
- `kubectl describe hpa sample-app-hpa -n autoscale-demo` (View HPA events and decisions)
- `kubectl get events -n autoscale-demo --sort-by='.lastTimestamp'` (View all namespace events chronologically)
