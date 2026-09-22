# demo-kubernetes

Manifeste für ein AWS EKS Cluster: AWS Load Balancer Controller, ArgoCD, App (`cicd/`, `gitops/`, `manual/`).

## Checkliste nach Cluster-Neustart

1. **Werte aktualisieren** (VPC/ACM-Zertifikat/RDS-Secret-ARN werden von Terraform neu erstellt):
   - `vpcId` in `aws-load-balancer-controller/values.yaml`
   - `alb.ingress.kubernetes.io/certificate-arn` in `cicd/ingress.yaml`, `gitops/ingress.yaml`, `manual/ingress.yaml`, `argocd/ingress.yaml`
   - `remoteRef.key` (RDS-Secret-ARN) in `manual/external-secret.yaml`, `gitops/external-secret.yaml`
   - `DB_SECRET_ARN` GitHub-Actions-Variable im Repo `demo-app` (`gh variable set DB_SECRET_ARN --repo t-foerst/demo-app --body "<arn>"`) — für `cicd/`

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
   → `gitops/` wird danach automatisch von ArgoCD deployed. `application-external-secrets.yaml` + `application-infrastructure.yaml` bringen den External Secrets Operator inkl. `ClusterSecretStore` mit, den `gitops/` und `manual/` beide über `ExternalSecret` nutzen (kein manuelles Secret nötig).

4. **`cicd/` und `manual/` deployen:**
   ```bash
   kubectl apply -f cicd/namespace.yaml
   kubectl apply -k cicd/
   kubectl apply -k manual/
   ```
   > `cicd/namespace.yaml` bewusst **nicht** Teil von `cicd/kustomization.yaml` und muss einmalig admin-seitig angelegt werden: die `github-actions-deploy`-IAM-Rolle hat nur `AmazonEKSEditPolicy` (namespaced, kein Namespace-Create/-Patch) — der CI-Workflow deployt danach nur noch in den bereits existierenden Namespace.

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
