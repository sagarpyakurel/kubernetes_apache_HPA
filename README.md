# kubernetes_apache_HPA — concise guide

Apache HTTP Server on Kubernetes with Horizontal Pod Autoscaling (HPA).

Quick — what this repo contains
- namespace.yaml : Namespace `apache`.
- deployment.yaml : Deployment `apache-deployment` (image: httpd:latest) with resource requests/limits (cpu: 100m / 200m, memory: 128Mi / 256Mi).
- service.yaml : ClusterIP service `apache-service`.
- hpa.yaml     : HPA `apache-hpa` (min:1, max:4, cpu target: averageUtilization: 4 in repo).
- config.yaml  : kind cluster config used by the author (optional).

Note: the HPA target in hpa.yaml is set to `averageUtilization: 4` (very low). For real workloads use a higher value (e.g., 40 or 50).

Prerequisites (minimal)
- kubectl configured to your cluster.
- metrics-server installed (HPA needs it for CPU metrics).

Deploy (one-liners)

```bash
# apply resources (namespace, deployment, service, hpa)
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml

# confirm
kubectl get ns,deploy,svc,hpa -n apache
```

Install metrics-server (quick)

1) Apply the official metrics-server manifest:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.1/components.yaml
```

2) Edit the metrics-server deployment to allow insecure kubelet TLS:

```bash
kubectl -n kube-system edit deployment metrics-server
# under containers[0].args add the line:
# - --kubelet-insecure-tls
# Save and exit the editor.
```

3) Restart metrics-server:

```bash
kubectl -n kube-system rollout restart deployment metrics-server
```

4) Verify metrics are available:

```bash
kubectl top nodes
kubectl top pods -A
```

Stress test (same approach you used)

1) Start BusyBox shell in the same namespace:

```bash
kubectl run load-generator -n apache --image=busybox -it --rm --restart=Never -- /bin/sh
```

2) Inside BusyBox run a loop that hits the service:

```sh
while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done
```

3) Watch scaling from another terminal:

```bash
kubectl get hpa -n apache -w
kubectl get pods -n apache -w
```

Expected behavior (from your testing)
- Under continuous load the HPA will increase replicas (you observed scaling up to 4 pods).
- After stopping the BusyBox loop, HPA will reduce replicas back to the minimum (1) over time.

Why resource requests & limits matter (short)
- requests: tells the scheduler how much CPU/memory the pod needs so it places pods correctly.
- limits: caps what a container can use so one pod won't starve others.
- HPA CPU% is calculated vs the requested CPU. If requests are missing or wrong, HPA decisions will be inaccurate.

Quick cleanup

```bash
kubectl delete -f hpa.yaml -n apache
kubectl delete -f service.yaml -n apache
kubectl delete -f deployment.yaml -n apache
kubectl delete -f namespace.yaml
```

Suggested small improvements (optional)
- Fix hpa.yaml averageUtilization to a realistic value (e.g., 40 or 50).
- Add an optional non-interactive load Job manifest for reproducible tests.

