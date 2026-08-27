# kubernetes_apache_HPA — concise guide

Apache HTTP Server on Kubernetes with Horizontal Pod Autoscaling (HPA).

Files in this repo (main)
- namespace.yaml — Namespace: `apache`
- deployment.yaml — Deployment `apache-deployment` (image: `httpd:latest`) with resources: requests 100m/128Mi, limits 200m/256Mi
- service.yaml — ClusterIP `apache-service` (port 80)
- hpa.yaml — HPA `apache-hpa` (min:1, max:4, averageUtilization: 50)
- load-job.yaml — non-interactive BusyBox load Job (reproducible)
- config.yaml — kind cluster config (optional)

Prerequisites
- kubectl configured to your cluster
- metrics-server installed (HPA needs CPU metrics)

Deploy (apply manifests)
```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl get ns,deploy,svc,hpa -n apache
```

Install metrics-server (quick)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.1/components.yaml
# then edit and add the kubelet TLS flag:
kubectl -n kube-system edit deployment metrics-server
# add under containers[0].args:
# - --kubelet-insecure-tls
kubectl -n kube-system rollout restart deployment metrics-server
kubectl top nodes
kubectl top pods -A
```

Test — method A: interactive BusyBox (your original)
Terminal A:
```bash
kubectl run load-generator -n apache --image=busybox -it --rm --restart=Never -- /bin/sh
# inside BusyBox:
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
# Ctrl+C to stop (exits and removes pod)
```
Terminal B: (observe)
```bash
kubectl get hpa -n apache -w
kubectl get pods -n apache -w
```

Test — method B: non-interactive Job (reproducible)
```bash
kubectl apply -f load-job.yaml -n apache
# stop and remove:
kubectl delete job apache-load-job -n apache
```

Observe expected behavior
- Under sustained load HPA increases replicas (you saw up to 4).
- After stopping load HPA scales back to 1 over time.

Why resource requests & limits (short)
- requests: reserve CPU/memory so scheduler places pods correctly.
- limits: cap usage to prevent noisy neighbors.
- HPA CPU% is calculated against requested CPU — set requests for correct scaling.

Quick cleanup
```bash
kubectl delete -f load-job.yaml -n apache  # if used
kubectl delete -f hpa.yaml -n apache
kubectl delete -f service.yaml -n apache
kubectl delete -f deployment.yaml -n apache
kubectl delete -f namespace.yaml
```

You can also do this (practical)
- Change HPA target: averageUtilization → 50 (done in this repo) to avoid aggressive scaling.
- Use the non-interactive load job for repeatable tests (load-job.yaml present).
- Optional extras: add readiness/liveness probes, add resource limits for load job, add monitoring stack (Prometheus/Grafana).

Notes
- The author used an EC2 node with ~20 GB storage for testing. Node CPU/RAM determine how many pods you can run — adjust requests/limits accordingly.

