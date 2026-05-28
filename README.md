<div align="center">
  <img src="https://kubernetes.io/images/favicon.png" width="100" />
  <h1>🚀 GKE Autoscaling Masterclass</h1>
  <p><em>A comprehensive, visual training guide for Kubernetes Autoscaling on Google Cloud</em></p>
  
  ![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
  ![Google Cloud](https://img.shields.io/badge/GoogleCloud-%234285F4.svg?style=for-the-badge&logo=google-cloud&logoColor=white)
</div>

---

## 📖 Overview

This project is designed as a **live demonstration** of Kubernetes autoscaling capabilities on Google Kubernetes Engine (GKE). It covers:
- 📈 **Horizontal Pod Autoscaler (HPA)**
- 🏢 **Cluster Autoscaler**
- 🔄 **Rolling Updates & Rollbacks**
- 🛡️ **Production Best Practices**

---

## 🏗️ 1. Cluster Provisioning

Run the following commands to provision a specialized GKE cluster optimized for this demonstration.

```bash
# Set your project (replace with your actual project ID)
# gcloud config set project YOUR_PROJECT_ID

# Create the demo cluster
gcloud container clusters create autoscale-demo-cluster \
  --region asia-south1 \
  --num-nodes 1 \
  --machine-type e2-standard-4 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 2 \
  --node-pool-name general-pool

# Retrieve cluster credentials
gcloud container clusters get-credentials autoscale-demo-cluster --region asia-south1
```

---

## 🧠 2. Architecture Deep Dive

Understanding the interplay between different scaling mechanisms is crucial.

### ⚖️ HPA vs Cluster Autoscaler
| Component | What it Monitors | Action Taken |
| :--- | :--- | :--- |
| **Horizontal Pod Autoscaler (HPA)** | CPU/Memory usage of Pods | Scales the number of **Pod Replicas** up or down based on load. |
| **Cluster Autoscaler** | Unscheduled (Pending) Pods | Provisions or removes **Worker Nodes** from the cloud provider (GCP). |

### 🔍 How Pending Pods Trigger Node Creation
When HPA scales up the number of replicas, the Kubernetes Scheduler tries to assign them to nodes. Because our application has resource `requests` (100m CPU), the node must have that much unallocated capacity. 
> 💡 **The Trigger:** Once the single node runs out of capacity, new Pods stay in a `Pending` state. The Cluster Autoscaler detects these pending pods and automatically provisions a new `e2-standard-4` node.

### 🔄 Zero-Downtime Rolling Updates
When you update a Deployment, Kubernetes doesn't kill all old Pods at once. It starts a new ReplicaSet, scales it up slightly, waits for the new Pods to pass their **Readiness probes**, and then scales down the old ReplicaSet. 

### 🛡️ Why Probes are Critical
- ❤️ **Liveness Probe**: Detects if a container is dead/hung. Triggers a restart.
- 🚦 **Readiness Probe**: Detects if a container is ready for traffic. Prevents routing to starting-up containers during rolling updates.

---

## 🎓 3. Visual Demo Flow

Follow these steps while presenting to the class for maximum impact.

### Step 1: Deploy the Foundation
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
# Watch pods, hpa, and nodes in real-time
watch -n 1 'kubectl get hpa,pods -n autoscale-demo; echo ""; kubectl get nodes'
```

### Step 2: Trigger Autoscaling 🚀
Explain that the application is running and monitored by the HPA. We will now generate load.

```bash
# Scale the load generator to start hitting the service hard
kubectl scale deployment load-generator -n autoscale-demo --replicas=20
```

### Step 3: Observe the Magic 🪄
*Narration:* "As the load generator sends traffic, the CPU utilization of our sample-app will rise. Once it crosses 50%, HPA will start adding new replicas."

**What to point out to students:**
1. HPA `TARGETS` percentage skyrocketing.
2. New `sample-app` Pods being created.
3. Pods entering a `Pending` state as the node fills up.
4. *Narration:* "Notice the Pending pods. This tells the Cluster Autoscaler we need more capacity. It will now spin up a second node."

> 💡 **Pro-Tip:** Run `kubectl get pods -n autoscale-demo -o wide` and point out the `NODE` column to show students that the new pods are being scheduled onto the newly provisioned node!

### Step 4: Perform a Rolling Update 🚢
Once the cluster has scaled out, demonstrate a zero-downtime deployment.

```bash
# Update the image to a newer version
kubectl set image deployment/sample-app app=nginx:1.25.4-alpine -n autoscale-demo

# Watch the rollout status
kubectl rollout status deployment/sample-app -n autoscale-demo
```
*Narration:* "Kubernetes is rolling out the new image. It creates new pods, waits for them to become 'Ready', and then terminates the old pods. Traffic flows uninterrupted."

### Step 5: Perform a Rollback ⏪
Show how to quickly recover from a bad deployment.

```bash
# Check rollout history and undo the last rollout
kubectl rollout history deployment/sample-app -n autoscale-demo
kubectl rollout undo deployment/sample-app -n autoscale-demo

# Verify restoration
kubectl rollout status deployment/sample-app -n autoscale-demo
```

### Step 6: Cleanup 🧹
```bash
kubectl scale deployment load-generator -n autoscale-demo --replicas=0
# Or delete the entire namespace: kubectl delete -f .
```

---

## 🛠️ Useful Monitoring Commands

| Command | Purpose |
| :--- | :--- |
| `kubectl get hpa -n autoscale-demo -w` | Watch HPA metrics dynamically |
| `kubectl get pods -n autoscale-demo -w` | Watch pod lifecycle transitions |
| `kubectl top pods -n autoscale-demo` | View CPU/Memory usage per pod |
| `kubectl top nodes` | View CPU/Memory usage per node |
| `kubectl describe hpa sample-app-hpa -n autoscale-demo`| View HPA events and decisions |
| `kubectl get events -n autoscale-demo --sort-by='.lastTimestamp'`| View all namespace events chronologically |

---
<div align="center">
  <i>Built with ❤️ for Kubernetes Training</i>
</div>
