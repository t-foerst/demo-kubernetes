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

3. **ArgoCD installieren** (nur Tooling, deployed die App noch nicht):
   ```bash
   kubectl apply -f argocd/namespace.yaml
   kubectl apply -f secrets/argocd-webhook-github-secret.yaml   # Webhook-Secret für Schritt 5, siehe unten
   helm repo add argo https://argoproj.github.io/argo-helm && helm repo update argo
   helm install argocd argo/argo-cd -n argocd -f argocd/values.yaml
   kubectl apply -f argocd/ingress.yaml
   kubectl apply -f argocd/application-external-secrets.yaml
   kubectl apply -f argocd/application-infrastructure.yaml
   ```
   `secrets/` ist gitignored (nur `*.example` liegt im Repo). Einmalig anlegen: `cp secrets/argocd-webhook-github-secret.yaml.example secrets/argocd-webhook-github-secret.yaml` und den Platzhalter durch einen festen Wert ersetzen (z. B. `openssl rand -hex 32`). `argocd/values.yaml` referenziert das Secret nur per `$argocd-webhook-github:secret`.

   Admin-Passwort: `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d`
   → `application-external-secrets.yaml` + `application-infrastructure.yaml` bringen den External Secrets Operator inkl. `ClusterSecretStore` mit, den `gitops/` und `manual/` beide über `ExternalSecret` nutzen (kein manuelles Secret nötig). `gitops/` selbst wird erst deployed, wenn `argocd/application-demo-app.yaml` angewendet wird (siehe Schritt 6).

4. **ALB-Hostname ermitteln und in Terraform (Cloudflare-DNS, Variable `alb_hostname`) eintragen** — alle vier Ingresses teilen sich einen ALB (`group.name: demo-cluster`), der ALB entsteht bereits durch `argocd/ingress.yaml` aus Schritt 3:

   | Host | Namespace |
   |---|---|
   | `cicd.foerst.haus` | `demo-app-cicd` |
   | `gitops.foerst.haus` | `demo-app-argocd` |
   | `manual.foerst.haus` | `demo-app-manual` |
   | `argocd.foerst.haus` | `argocd` |

   ```bash
   kubectl get ingress -n argocd argocd-server
   ```

5. **GitHub-Webhook für ArgoCD** — Push auf `demo-kubernetes` löst sofort einen Refresh aus statt erst nach dem Poll-Intervall (~3 min). **Nur einmalig nötig**, nicht nach jedem Cluster-Neustart: Secret (fester Wert aus `secrets/`) und Hook-URL bleiben gleich, der Hook in GitHub überdauert den Cluster (Zustellungen schlagen nur fehl, solange der Cluster aus ist). Wert aus `secrets/argocd-webhook-github-secret.yaml` übernehmen:
   ```bash
   gh api repos/t-foerst/demo-kubernetes/hooks -f name=web -f 'events[]=push' \
     -f 'config[url]=https://argocd.foerst.haus/api/webhook' -f 'config[content_type]=json' \
     -f "config[secret]=$(kubectl get secret argocd-webhook-github -n argocd -o jsonpath='{.data.secret}' | base64 -d)"
   ```
   Prüfen: GitHub → `demo-kubernetes` → Settings → Webhooks → Recent Deliveries (HTTP 200) bzw. `kubectl logs -n argocd deploy/argocd-server | grep -i webhook`.
   > Für Messungen in der Standardkonfiguration ohne Webhook (Bachelorarbeit, Variante B1) den Hook deaktivieren statt löschen: `gh api -X PATCH repos/t-foerst/demo-kubernetes/hooks/<id> -F active=false` (mit `-F active=true` wieder an; ID per `gh api repos/t-foerst/demo-kubernetes/hooks --jq '.[].id'`). ArgoCD fällt dann auf das Polling zurück.

6. **App deployen (optional)** — drei unabhängige Wege, keiner ist Voraussetzung für einen anderen:
   ```bash
   # Manual
   kubectl apply -k manual/

   # CI/CD (Namespace einmalig admin-seitig vorbereiten, s.u.)
   kubectl apply -f cicd/namespace.yaml
   kubectl apply -k cicd/

   # ArgoCD/GitOps
   kubectl apply -f argocd/application-demo-app.yaml
   ```
   > `cicd/namespace.yaml` bewusst **nicht** Teil von `cicd/kustomization.yaml` und muss einmalig admin-seitig angelegt werden: die `github-actions-deploy`-IAM-Rolle hat nur `AmazonEKSEditPolicy` (namespaced, kein Namespace-Create/-Patch) — der CI-Workflow deployt danach nur noch in den bereits existierenden Namespace.

   Ausführliche Anleitung inkl. Voraussetzungen je Weg: Obsidian-Vault, `Studium/Kubernetes Cluster (Uni) - Anleitung.md`.
