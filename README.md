# demo-kubernetes

Manifeste für ein AWS EKS Cluster

> **Bei jedem Neu-Hochfahren des Clusters:** `vpcId` in `aws-load-balancer-controller/values.yaml` neu setzen (VPC wird von Terraform neu erstellt). Ebenso `alb.ingress.kubernetes.io/certificate-arn` in `cicd/ingress.yaml`, `gitops/ingress.yaml`, `manual/ingress.yaml` und `argocd/ingress.yaml` auf die neue ACM-Cert-ARN anpassen.

## 1. AWS Load Balancer Controller installieren

```bash
kubectl apply -f aws-load-balancer-controller/service-account.yaml

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f aws-load-balancer-controller/values.yaml
```

## 2. App deployen

Die App gibt es dreifach, gleicher Code, drei getrennte Deploy-Wege/Namespaces:

- **`cicd/`** → Namespace `demo-app-cicd`, klassisch per CI/CD-Pipeline deployed (`kubectl apply -k cicd/`)
- **`gitops/`** → Namespace `demo-app-argocd`, von ArgoCD verwaltet (Abschnitt 3), kein manuelles Apply nötig
- **`manual/`** → Namespace `demo-app-manual`, für Ad-hoc/manuelle Deploys (Abschnitt 5)

Alle drei Wege nutzen dieselben Deployment/Service/Ingress-Definitionen und identisches Secret-Management über External Secrets Operator (siehe Abschnitt 5).

```bash
kubectl apply -k cicd/
```

Der `demo-app-argocd`-Namespace wird nicht manuell deployed, sondern über die ArgoCD-Application aus Abschnitt 3 synchronisiert.

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
kubectl apply -f argocd/application-external-secrets.yaml
kubectl apply -f argocd/application-infrastructure.yaml
```

Initiales Admin-Passwort:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

`argocd/application-demo-app.yaml` synct `gitops/` automatisch (`prune` + `selfHeal`) nach `demo-app-argocd`.

## 4. ALB-Hostname in Terraform eintragen

Alle vier Ingresses (`demo-app-cicd`, `demo-app-argocd`, `demo-app-manual`, `argocd`) teilen sich über `alb.ingress.kubernetes.io/group.name: demo-cluster` denselben ALB und werden per Hostname unterschieden:

| Host                  | Namespace          | Ingress         |
|------------------------|--------------------|------------------|
| `cicd.foerst.haus`     | `demo-app-cicd`    | `demo-app`       |
| `gitops.foerst.haus`   | `demo-app-argocd`  | `demo-app`       |
| `manual.foerst.haus`   | `demo-app-manual`  | `demo-app`       |
| `argocd.foerst.haus`   | `argocd`           | `argocd-server`  |

Nach dem Deployment den (gemeinsamen) ALB-Hostnamen ermitteln und für alle vier Hosts in Terraform (z. B. als Route53-Records) eintragen:

```bash
kubectl get ingress -n argocd argocd-server
```

## 5. Manuelles Deployment & Secret-Management

`manual/` ist für Ad-hoc-Deploys gedacht (z. B. lokales Testen, Debugging) — bewusst **ohne** Abhängigkeit von ArgoCD/External Secrets Operator, damit es auch funktioniert wenn diese (noch) nicht laufen. Anders als `gitops/` nutzt `manual/` daher kein `ExternalSecret`, sondern ein ganz normales, lokal erzeugtes `Secret`:

```bash
cp manual/secret.example.yaml manual/secret.yaml
# DB_PASSWORD in manual/secret.yaml eintragen (z. B. aus AWS Secrets Manager Console/CLI holen)
kubectl apply -f manual/secret.yaml
kubectl apply -k manual/
```

**Secret-Management:** `manual/secret.yaml` wird lokal aus der Vorlage `manual/secret.example.yaml` erzeugt und ist über `.gitignore` von Git ausgeschlossen — es landet also nie im Repo. Das ist bewusst simpel gehalten: keine ClusterSecretStore-, ESO- oder IRSA-Abhängigkeit, nur ein Secret das einmalig mit dem echten Wert befüllt und applied wird. Nachteil gegenüber `gitops/`: keine automatische Rotation, der Wert muss bei Änderung manuell aktualisiert werden.

## 6. High Availability & Autoscaling

Alle drei App-Deployments (`cicd/deployment.yaml`, `gitops/deployment.yaml`, `manual/deployment.yaml`) sind für HA über die zwei EKS-Worker-Nodes (je einer pro AZ) ausgelegt:

- **`topologySpreadConstraints`** (Key `topology.kubernetes.io/zone`, `whenUnsatisfiable: DoNotSchedule`) verteilt die Pods zwingend auf beide AZs/Nodes, statt beide auf denselben Node zu packen.
- **`PodDisruptionBudget`** (`minAvailable: 1`) verhindert, dass bei Node-Drain/-Wartung beide Pods gleichzeitig entfernt werden.
- **`HorizontalPodAutoscaler`** (`minReplicas: 2`, `maxReplicas: 4`, CPU-Ziel 70 %) skaliert bei Last zusätzliche Pods innerhalb der bestehenden zwei Nodes hoch (kein Node-Autoscaling nötig).

Voraussetzung für die HPA ist ein laufender `metrics-server` im Cluster (liefert die CPU-Metriken); auf EKS ggf. als Addon oder per Helm-Chart nachinstallieren, falls noch nicht vorhanden.

**GitOps-Hinweis:** Da der HPA `spec.replicas` des Deployments verändert, ignoriert die ArgoCD-Application (`argocd/application-demo-app.yaml`) dieses Feld per `ignoreDifferences`, damit `selfHeal` die HPA-Skalierung nicht laufend zurücksetzt. In `demo-app-cicd` und `demo-app-manual` gibt es diesen Schutz nicht — jedes erneute `kubectl apply -k` setzt `replicas` kurzzeitig auf den in Git hinterlegten Wert zurück, bis der HPA erneut hochskaliert.
