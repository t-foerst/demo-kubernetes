# demo-kubernetes

Manifeste für ein AWS EKS Cluster

## 1. AWS Load Balancer Controller installieren

```bash
kubectl apply -f aws-load-balancer-controller/service-account.yaml
```

Anschließend den Controller per Helm installieren:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f aws-load-balancer-controller/values.yaml
```

## 2. App deployen

```bash
kubectl apply -k app/
```

Das erstellt:
- `namespace.yaml` – Namespace `demo-app`
- `deployment.yaml` – 2 Replicas, Image `ghcr.io/t-foerst/demo-app:latest`, Port 80
- `service.yaml` – ClusterIP Service
- `ingress.yaml` – ALB Ingress (internet-facing, target-type `ip`)

Nach dem Deploy die ALB-URL ermitteln:

```bash
kubectl get ingress -n demo-app demo-app
```
