# Kubernetes Project: Apache HTTP Server with Horizontal Pod Autoscaling (HPA)

This repository demonstrates running an Apache HTTP Server on Kubernetes with Horizontal Pod Autoscaling (HPA). This README gives a beginner-friendly, step-by-step guide to deploy the app, configure resource requests/limits, run the HPA, and stress-test the application using BusyBox.

---

## Overview

- Goal: Run Apache on Kubernetes, configure resource requests/limits, enable HPA so the deployment scales under load, and verify scaling with a simple load generator.
- Test environment used by the author: an EC2 instance sized with 20 GB storage (ensure your Kubernetes node(s) have enough CPU and memory for pods).

---

## Prerequisites

1. A Kubernetes cluster (any managed cluster like EKS/GKE/AKS or a single-node cluster). Ensure kubectl is configured to talk to the cluster.
2. kubectl installed and authenticated (kubectl version >= 1.18 recommended).
3. metrics-server or other resource metrics provider installed in cluster (HPA uses metrics-server to read CPU/memory). Example: install metrics-server if not present.
4. Basic tools: wget, busybox image will be used as a load generator.

Tip: The author used an EC2 instance with 20 GB storage to host the cluster/node running the app — storage size matters for logs and images but CPU/RAM on nodes determine how many pods you can run.

---

## Quick-start — deploy Apache, expose it, and enable HPA

These quick commands will create a simple deployment, expose it with a ClusterIP service, and create an HPA that scales based on CPU usage.

1) Create the Apache deployment:

```bash
kubectl create deployment apache --image=httpd:2.4
```

2) Expose it as a ClusterIP service so other pods in the cluster can reach it (service name `apache-service`):

```bash
kubectl expose deployment apache --port=80 --target-port=80 --name=apache-service
```

3) Add resource requests and limits to the deployment (recommended). Example: patch the deployment with requests/limits (this sets a small example — tune to your environment):

```bash
kubectl patch deployment apache --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/resources","value":{
    "requests":{"cpu":"100m","memory":"128Mi"},
    "limits":{"cpu":"250m","memory":"256Mi"}
  }}
]'
```

Why requests and limits? Short, direct reasons:
- requests: tells the scheduler how much CPU/memory the pod needs to be placed on a node (prevents scheduling too many pods on the same node).
- limits: caps the maximum resources a pod can use (prevents one pod from using all node resources and affecting others).
- HPA behavior: HPA uses CPU (or custom metrics). If requests are missing, CPU utilization percentages may be misleading — set requests so HPA calculations are meaningful.

4) Create an HPA that scales the apache deployment automatically (example target CPU utilization 50%, min 1 pod, max 5 pods):

```bash
kubectl autoscale deployment apache --cpu-percent=50 --min=1 --max=5
```

Check HPA and deployment status:

```bash
kubectl get hpa
kubectl get deployment apache
kubectl get pods -l app=apache
```

Note: `kubectl top pods` will show live CPU usage if metrics-server is installed.

---

## Stress testing the application (your BusyBox approach)

Below is the recommended beginner-friendly exact sequence you used, cleaned up and explained.

1) Start a BusyBox pod as a temporary load generator (runs an interactive shell and will be removed when you exit):

```bash
kubectl run load-generator --image=busybox -it --rm --restart=Never -- /bin/sh
```

2) Inside the BusyBox shell, run this loop to continuously hit the service. Replace the service host if your namespace is different; this example assumes the `apache` Service is in namespace `apache`. If you deployed in `default`, use `apache-service.default.svc.cluster.local` or just `apache-service`.

```sh
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

Notes:
- `wget -q -O-` requests the page and prints the HTML to stdout; `-q` quiets wget logs so the loop is cleaner.
- Press Ctrl+C inside the BusyBox shell to stop the loop; exiting the shell will remove the pod because of `--rm`.

3) Observe scaling:
- In another terminal, run:

```bash
kubectl get hpa -w
kubectl get pods -l app=apache -w
```

- While the load loop runs you should see the HPA increase the number of pods (you observed it scaled to 4 pods). When you stop the busybox loop, HPA will reduce the pod count back down (you observed it returning to 1 pod after load stopped).

Why this works: HPA watches CPU usage (or other metrics). Under continuous requests, CPU usage climbs, HPA increases replicas. When load stops, utilization drops and HPA scales down.

---

## Monitoring and debugging commands

- Show HPA details and current metrics:

```bash
kubectl describe hpa
```

- Show current pods and their resource usage (requires metrics-server):

```bash
kubectl top pods
```

- See deployment events and rollout status:

```bash
kubectl rollout status deployment/apache
kubectl describe deployment apache
```

---

## Resource requests & limits — short and direct (why required)

- Requests = the amount of resource the scheduler uses to decide where to place a pod.
- Limits = the maximum the container can consume; prevents noisy-neighbor problems.
- HPA percent targets are calculated against the requested CPU. If you don't set requests, HPA scaling decisions may be wrong.
- Set realistic values: start small (example requests: cpu=100m, memory=128Mi) and increase if you see throttling or OOMs.

Quick rule of thumb:
- Small web server: requests 100m CPU / 128Mi memory; limits 250m CPU / 256Mi memory.
- Increase both if your server handles more traffic or uses more memory.

---

## Tuning tips

- HPA target CPU: lower numbers (e.g., 30-50) scale earlier; higher numbers (70+) wait for heavier load.
- Min/max replicas: set min to 1 or greater depending on availability requirements; max to the number your cluster nodes can support.
- Stabilization window & scale policies: use HPA v2 and advanced options for smoother scaling if needed.

---

## Cleanup

To remove the resources created in this quick-start:

```bash
kubectl delete hpa apache
kubectl delete service apache-service
kubectl delete deployment apache
```

---

## Troubleshooting

- If HPA never scales: ensure metrics-server is installed and `kubectl top pods` returns data.
- If pods are Pending: check node resources and the `requests` values — you may need larger nodes.
- If pods are OOMKilled: increase memory requests/limits.

---

If you want, I can also:
- Add example YAML manifest files (Deployment, Service, HPA) in the repo for one-click `kubectl apply -f` usage.
- Add instructions for installing metrics-server if it's not present in your cluster.

