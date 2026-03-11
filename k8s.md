# Kontekst: K8s Security na GCP — podsumowanie dla Claude Code

## Cel projektu
Wdrożenie monitoringu bezpieczeństwa Kubernetes z użyciem narzędzi open-source. Dwie kluczowe warstwy do pokrycia: **podatności (CVE)** i **runtime security**.

---

## Stack docelowy (trzy narzędzia, każde inne zadanie)

### 1. Trivy Operator — podatności CVE
- Ciągłe skanowanie obrazów, manifestów YAML, RBAC, secretów
- Wyniki jako Kubernetes CRDs (`VulnerabilityReport`, `ConfigAuditReport`, etc.)
- Auto-reskan przy każdej zmianie workloadu
- Eksport metryk do Prometheus → Grafana

### 2. Falco — runtime detekcja
- eBPF-based, wykrywa anomalie w czasie rzeczywistym
- Tylko detekcja (alerty), nie blokuje
- Overhead ~5–10% CPU
- Alerty przez Falcosidekick → Slack / Prometheus / SIEM

### 3. Tetragon — runtime enforcement
- eBPF kernel hooks, overhead <1% CPU
- Wykrywa I blokuje (SIGKILL) — np. shell w prod podzie
- Natively K8s-aware (labels, namespaces)
- Działa dobrze razem z Falco (Falco=detekcja, Tetragon=enforcement)

### 4. Kubescape Operator — compliance & posture
- Najlepszy w klasie do compliance: CIS Benchmark, NSA-CISA, MITRE ATT&CK, ISO 27001, DORA
- RBAC visualizer, misconfiguracje, network policy recommendations
- Wyniki jako CRDs, eksport do Prometheus
- Nie zastępuje Trivy (słabszy w CVE) ani Falco (brak runtime)

---

## Środowisko testowe w GCP

### Klaster GKE Standard (NIE Autopilot — za dobrze zabezpieczony)
```bash
gcloud container clusters create kubescape-vulnerable-lab \
  --region=europe-central2 \
  --num-nodes=1 \
  --machine-type=e2-small \
  --no-enable-shielded-nodes \
  --no-enable-network-policy \
  --metadata disable-legacy-endpoints=false

gcloud container clusters get-credentials kubescape-vulnerable-lab \
  --region=europe-central2
```

### Podatne workloady do testów (vulnerable-workloads.yaml)
Celowo złe konfiguracje gwarantujące findings w Kubescape/Trivy:
- Privileged container (C-0017 — Critical)
- HostPath mount na `/` (C-0045 — High)
- Wildcard RBAC ClusterRole `*/*/*` (C-0034 — High)
- Brak resource limits (C-0009 — Medium)
- Hardcoded secrets w env vars (C-0012 — Medium)
- Brak securityContext (C-0044 — Medium)

```bash
kubectl apply -f vulnerable-workloads.yaml
```

---

## Instalacja narzędzi (Helm)

### Trivy Operator
```bash
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system --create-namespace \
  --set trivy.ignoreUnfixed=true \
  --set trivy.severity=HIGH,CRITICAL \
  --set operator.scanJobsConcurrentLimit=5

# Sprawdzenie wyników
kubectl get vulnerabilityreports -A
kubectl get configauditreports -A
kubectl get rbacassessmentreports -A
```

### Kubescape Operator
```bash
helm repo add kubescape https://kubescape.github.io/helm-charts/
helm install kubescape kubescape/kubescape-operator \
  --namespace kubescape --create-namespace \
  --set clusterName=$(kubectl config current-context) \
  --set kubescape.scanSchedule="0 2 * * *"

# Skanowanie manualne
kubescape scan framework nsa --enable-color
kubescape scan framework mitre --enable-color
kubescape scan -f html -o report.html
```

### Falco
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falco.json_output=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.prometheus.hostport="prometheus-server:9093"
```

### Tetragon
```bash
helm repo add cilium https://helm.cilium.io
helm install tetragon cilium/tetragon \
  --namespace kube-system

# Przykładowa polityka: blokada shell w namespace production
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-shell-in-prod
spec:
  kprobes:
  - call: "sys_execve"
    syscall: true
    args:
    - index: 0
      type: "string"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Postfix"
        values: ["/bin/sh", "/bin/bash"]
      matchNamespaces:
      - namespace: production
      matchActions:
      - action: Sigkill
EOF
```

---

## Architektura multi-cluster (enterprise)

```
Każdy klaster:
  ├── trivy-operator     (namespace: trivy-system)
  ├── kubescape-operator (namespace: kubescape)
  ├── falco + sidekick   (namespace: falco)
  └── tetragon           (namespace: kube-system)
         │
         ▼ metryki / logi
Centralny stack:
  ├── Prometheus (agregacja metryk)
  ├── Loki (logi z Falco)
  └── Grafana (dashboardy + alerty)
```

### Skalowanie Kubescape na dużych klastrach
Domyślnie 500 MiB RAM / 500m CPU — dobre do ~1250 zasobów.
Powyżej: +100 MiB RAM na każde 200 zasobów ponad próg.

```bash
helm upgrade kubescape kubescape/kubescape-operator \
  --set kubescape.resources.requests.memory="1Gi" \
  --set kubescape.resources.limits.memory="2Gi" \
  --set nodeAgent.multipleDaemonSets=true  # dla heterogenicznych node pools
```

---

## Kolejność wdrożenia (rekomendowana)

| Etap | Co | Kiedy |
|------|-----|-------|
| 1 | Trivy Operator + Grafana | Tydzień 1 — natychmiastowa widoczność CVE |
| 2 | Kubescape Operator | Tydzień 1-2 — compliance i posture |
| 3 | Falco + Falcosidekick | Tydzień 2-3 — runtime detekcja |
| 4 | Tetragon (tryb audit) | Tydzień 3-4 — obserwacja bez blokowania |
| 5 | Tetragon enforcement | Tydzień 4+ — stopniowe włączanie blokowania |

---

## Cleanup po testach (ważne — koszty GCP!)
```bash
gcloud container clusters delete kubescape-vulnerable-lab \
  --region=europe-central2
```

Koszt klastra testowego: ~$0.10-0.15/h (e2-small, 1 node).

---

## Notatki dodatkowe
- **ARMO Platform** — komercyjny SaaS/on-prem zbudowany na Kubescape; opcja jeśli potrzebne GUI bez własnego Grafana stacku
- **GKE Autopilot** — zbyt dobrze zabezpieczony do testów podatności, używaj GKE Standard