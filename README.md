# demo-kubernetes

Manifeste für ein AWS EKS Cluster

## 1. AWS Load Balancer Controller installieren

```bash
kubectl apply -f aws-load-balancer-controller/service-account.yaml

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f aws-load-balancer-controller/values.yaml
```

> **Bei jedem Neu-Hochfahren des Clusters:** `vpcId` in `aws-load-balancer-controller/values.yaml` neu setzen (VPC wird von Terraform neu erstellt). Ebenso `alb.ingress.kubernetes.io/certificate-arn` in `app/base/ingress.yaml` und `argocd/ingress.yaml` auf die neue ACM-Cert-ARN anpassen.

## 2. App deployen

`app/base/` (Deployment, Service, Ingress) wird per Kustomize-Overlay in zwei Namespaces ausgerollt:

- `app/overlays/argocd/` → Namespace `demo-app-argocd`, von ArgoCD verwaltet (Abschnitt 3)
- `app/overlays/cicd/` → Namespace `demo-app-cicd`, klassisch per CI/CD-Pipeline deployed

```bash
kubectl apply -k app/overlays/cicd
```

Der `argocd`-Namespace wird nicht manuell deployed, sondern über die ArgoCD-Application aus Abschnitt 3 synchronisiert.

## 3. ArgoCD installieren

```bash
kubectl apply -f argocd/namespace.yaml

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo

helm install argocd argo/argo-cd \
  -n argocd \
  -f argocd/values.yaml

kubectl apply -f argocd/ingress.yaml
kubectl apply -f argocd/application-demo-app.yaml
```

Initiales Admin-Passwort:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

`argocd/application-demo-app.yaml` synct `app/overlays/argocd` automatisch (`prune` + `selfHeal`) nach `demo-app-argocd`.

## 4. ALB-Hostname in Terraform eintragen

Alle drei Ingresses (`demo-app-cicd`, `demo-app-argocd`, `argocd`) teilen sich über `alb.ingress.kubernetes.io/group.name: demo-cluster` denselben ALB und werden per Hostname unterschieden:

| Host                  | Namespace          | Ingress         |
|------------------------|--------------------|------------------|
| `cicd.foerst.haus`     | `demo-app-cicd`    | `demo-app`       |
| `gitops.foerst.haus`   | `demo-app-argocd`  | `demo-app`       |
| `argocd.foerst.haus`   | `argocd`           | `argocd-server`  |

Nach dem Deployment den (gemeinsamen) ALB-Hostnamen ermitteln und für alle drei Hosts in Terraform (z. B. als Route53-Records) eintragen:

```bash
kubectl get ingress -n argocd argocd-server
```
