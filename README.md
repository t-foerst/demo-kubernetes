# demo-kubernetes

Manifeste für ein AWS EKS Cluster: AWS Load Balancer Controller, ArgoCD, App (`cicd/`, `gitops/`, `manual/`).

## Checkliste nach Cluster-Neustart

1. **Werte aktualisieren** (VPC/ACM-Zertifikat werden von Terraform neu erstellt):
   - `vpcId` in `aws-load-balancer-controller/values.yaml`
   - `alb.ingress.kubernetes.io/certificate-arn` in `cicd/ingress.yaml`, `gitops/ingress.yaml`, `manual/ingress.yaml`, `argocd/ingress.yaml`

2. **AWS Load Balancer Controller + metrics-server installieren:**
   ```bash
   kubectl apply -f aws-load-balancer-controller/service-account.yaml
   helm repo add eks https://aws.github.io/eks-charts && helm repo update eks
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system -f aws-load-balancer-controller/values.yaml
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

3. **ArgoCD installieren:**
   ```bash
   kubectl apply -f argocd/namespace.yaml
   helm repo add argo https://argoproj.github.io/argo-helm && helm repo update argo
   helm install argocd argo/argo-cd -n argocd -f argocd/values.yaml
   kubectl apply -f argocd/ingress.yaml
   kubectl apply -f argocd/application-demo-app.yaml
   kubectl apply -f argocd/application-external-secrets.yaml
   kubectl apply -f argocd/application-infrastructure.yaml
   ```
   Admin-Passwort: `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d`
   → `gitops/` wird danach automatisch von ArgoCD deployed.

4. **`cicd/` und `manual/` deployen:**
   ESO-Deployen (für manual)
   ```bash
   kubectl apply -k cicd/

   kubectl apply -k manual/
   ```

5. **ALB-Hostname ermitteln und in Terraform (Route53) eintragen** — alle vier Ingresses teilen sich einen ALB (`group.name: demo-cluster`):

   | Host | Namespace |
   |---|---|
   | `cicd.foerst.haus` | `demo-app-cicd` |
   | `gitops.foerst.haus` | `demo-app-argocd` |
   | `manual.foerst.haus` | `demo-app-manual` |
   | `argocd.foerst.haus` | `argocd` |

   ```bash
   kubectl get ingress -n argocd argocd-server
   ```
