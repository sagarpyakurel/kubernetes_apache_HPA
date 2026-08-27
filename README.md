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
This repository includes `load-job.yaml` so you can run a non-interactive, repeatable load generator instead of the interactive BusyBox.

1) Apply the load Job (creates a pod that continuously requests the service):
```bash
kubectl apply -f load-job.yaml -n apache
```

2) Inspect the Job and its Pod(s):
```bash
kubectl get job apache-load-job -n apache
kubectl get pods -n apache -l job-name=apache-load-job
# to stream logs from the load pod (replace POD_NAME):
kubectl logs -n apache -f POD_NAME
```

Notes about the Job
- The Job runs an infinite loop (the pod will keep making requests). Stop it by deleting the Job (see cleanup below).
- `backoffLimit: 0` is set so the Job won't retry on failure; it's intended as a continuously-running load generator.
- The BusyBox image used contains `wget` for simple HTTP requests. If your cluster restricts images, replace with another small tool image.
- You can adjust the request rate by changing or removing `sleep 0.1` in `load-job.yaml`.

Verification — observe HPA and pods
- Watch the HPA scale-up under sustained load:
```bash
kubectl get hpa -n apache -w
kubectl describe hpa/apache-hpa -n apache
```
- Watch pods and resource usage:
```bash
kubectl get pods -n apache -w
kubectl top pods -n apache
kubectl top nodes
```
Expected behavior
- Under sustained load the HPA should increase replicas (up to the `maxReplicas` set in `hpa.yaml`, you may see up to 4).
- After you delete the load Job, the HPA should scale back down to the `minReplicas` (1) over time as CPU usage falls.

Quick cleanup
```bash
kubectl delete -f load-job.yaml -n apache  # stop/remove the load generator
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
