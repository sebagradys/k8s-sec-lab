# K8s Security Lab — Raport z testów

**Data:** 2026-03-11
**Środowisko:** GKE Standard, europe-west1-b, e2-standard-2
**Klaster:** kubescape-vulnerable-lab

---

## Kontekst i cel

Celem tego labu było praktyczne przetestowanie open-source'owych narzędzi do monitorowania bezpieczeństwa Kubernetes — sprawdzenie jak działają, co wykrywają i gdzie są granice każdego z nich. Postawiliśmy celowo dziurawy klaster i zobaczyliśmy co narzędzia faktycznie potrafią wychwycić — zarówno na poziomie statycznej konfiguracji, jak i w czasie rzeczywistym.

### Co chcieliśmy sprawdzić

Bezpieczeństwo K8s można podzielić na dwie warstwy, które wymagają różnych narzędzi:

1. **Podatności i misconfiguracje** — rzeczy które są złe już w momencie deploymentu: kontenery z uprawnieniami roota, sekrety zakodowane na twardo w env vars, zbyt szerokie uprawnienia RBAC. To wykrywamy statycznie, bez uruchamiania ataków.

2. **Runtime security** — co się dzieje gdy klaster już działa: czy ktoś próbuje czytać `/etc/shadow`, uruchamia shelle w kontenerach, skanuje sieć. To wymaga ciągłego monitorowania syscalli.

Do każdej warstwy dobraliśmy inne narzędzia i sprawdziliśmy jak się uzupełniają.

### Docelowy stack (i dlaczego akurat te narzędzia)

Wszystkie narzędzia są open-source, działają jako operatory/daemony w samym klastrze i nie wymagają wysyłania danych na zewnątrz — ważne przy środowiskach z danymi wrażliwymi.

| Narzędzie | Warstwa | Co robi |
|-----------|---------|---------|
| **Trivy Operator** | Statyczna | Skanuje obrazy kontenerów pod kątem CVE, audytuje manifesty YAML i RBAC. Wyniki zapisuje jako K8s CRDs. |
| **Kubescape** | Statyczna | Sprawdza zgodność z frameworkami bezpieczeństwa: CIS Benchmark, NSA-CISA, MITRE ATT&CK. Daje score i listę konkretnych naruszeń. |
| **Falco** | Runtime | Monitoruje syscalle przez eBPF i alarmuje gdy coś wygląda podejrzanie — np. odczyt pliku z hasłami, uruchomienie shella w kontenerze, połączenie z API K8s. Tylko detekcja, nie blokuje. |
| **Tetragon** | Runtime | Też eBPF, ale z możliwością enforcement — może wysłać SIGKILL do procesu zanim cokolwiek zrobi. Działa na poziomie kernela, overhead poniżej 1% CPU. |

Falco i Tetragon świetnie działają razem: Falco alarmuje o wszystkim co podejrzane, Tetragon blokuje tylko najbardziej krytyczne rzeczy (np. shell w podzie produkcyjnym).

### Podejście do testów

Żeby narzędzia miały co wykrywać, celowo wdrożyliśmy zestaw workloadów z klasycznymi błędami konfiguracji — takimi które regularnie pojawiają się w prawdziwych klastrach. Nie było sensu testować na "czystej" konfiguracji, bo większość alertów by się nie pojawiła. Chcieliśmy zobaczyć konkretne findings, nie ogólne "narzędzie działa".

Na koniec wygenerowaliśmy też ręcznie podejrzane zdarzenia — czytanie plików z hasłami, szukanie kluczy prywatnych, próby uruchomienia shella — żeby przetestować runtime detection.

---

## Stack narzędzi

| Narzędzie | Wersja | Rola |
|-----------|--------|------|
| Trivy Operator | 0.30.0 | Skanowanie CVE + misconfig |
| Kubescape Operator | 1.30.5 | Compliance (CIS, NSA-CISA, MITRE) |
| Falco + Falcosidekick | 0.43.0 | Runtime detekcja anomalii |
| Tetragon | 1.6.0 | Runtime enforcement (SIGKILL) |

---

## Infrastruktura

Do testów potrzebowaliśmy klastra GKE Standard (nie Autopilot — ten jest zbyt dobrze zabezpieczony i blokuje część konfiguracji których chcieliśmy użyć). Celowo wyłączyliśmy shielded nodes i network policy, żeby mieć pełną kontrolę nad tym co wdrażamy.

**Finalna konfiguracja klastra:**
- 1 node, `e2-standard-2` (2 vCPU, 8 GB RAM), node image: `UBUNTU_CONTAINERD`
- Strefa: `europe-west1-b`
- Dysk: 50 GB SSD per node

Dojście do tej konfiguracji wymagało kilku iteracji. Zaczęliśmy od `e2-small` (2 vCPU, 2 GB), ale okazało się że cztery operatory bezpieczeństwa działające jednocześnie zużywają znacznie więcej zasobów niż można by się spodziewać — każdy z nich definiuje własne `resource requests`, przez co scheduler K8s nie miał gdzie postawić nowych podów i zostawały w stanie `Pending`. Przeszliśmy przez `e2-medium` (2 vCPU, 4 GB) który też okazał się za mały, i finalnie wylądowaliśmy na `e2-standard-2`. Szczegóły w sekcji "Znane problemy".

Dodatkowo domyślny node image GKE (Container-Optimized OS, COS) okazał się niekompatybilny z Tetragonem ze względu na bug w BTF kernela — konieczna była zmiana na `UBUNTU_CONTAINERD`. Więcej w sekcji Tetragon.

### Tworzenie klastra

```bash
# Włącz GKE API w projekcie (jeśli jeszcze nie włączone)
gcloud services enable container.googleapis.com

# Utwórz klaster
gcloud container clusters create kubescape-vulnerable-lab \
  --zone=europe-west1-b \
  --num-nodes=1 \
  --machine-type=e2-standard-2 \
  --no-enable-shielded-nodes \
  --no-enable-network-policy \
  --metadata disable-legacy-endpoints=false

# Podłącz kubectl
gcloud container clusters get-credentials kubescape-vulnerable-lab \
  --zone=europe-west1-b
```

### Node pool z Ubuntu (wymagany przez Tetragon)

Domyślny node image GKE (COS) jest niekompatybilny z Tetragonem — konieczne jest użycie `UBUNTU_CONTAINERD`. Najprościej stworzyć klaster od razu z Ubuntu, albo dodać node pool i usunąć domyślny COS pool.

```bash
# Dodaj Ubuntu node pool
gcloud container node-pools create pool-ubuntu \
  --cluster=kubescape-vulnerable-lab \
  --zone=europe-west1-b \
  --machine-type=e2-standard-2 \
  --num-nodes=1 \
  --disk-size=50 \
  --image-type=UBUNTU_CONTAINERD

# Usuń domyślny COS pool (po migracji podów)
gcloud container node-pools delete default-pool \
  --cluster=kubescape-vulnerable-lab \
  --zone=europe-west1-b
```

---

## Podatne workloady

Zamiast testować na realnych aplikacjach, wdrożyliśmy zestaw deploymentów które świadomie łamią podstawowe zasady bezpieczeństwa K8s. Każdy z nich odpowiada konkretnemu błędowi który regularnie pojawia się w prawdziwych klastrach — często przez nieuwagę lub kopiowanie konfiguracji z internetu bez sprawdzenia co ona robi.

Cel był taki, żeby każde narzędzie miało gwarantowane findings do wykrycia — bez tego testy nie miałyby sensu.

Plik: `vulnerable-workloads.yaml`

```bash
kubectl apply -f vulnerable-workloads.yaml
```

### Co zostało wdrożone

| Deployment | Celowa podatność | ID kontrolki |
|------------|-----------------|--------------|
| `privileged-container` | Privileged container | C-0017 / AVD-KSV-0017 |
| `hostpath-mount` | HostPath mount na `/` | C-0045 / AVD-KSV-0121 |
| `hardcoded-secrets` | Sekrety w env vars | C-0012 |
| `run-as-root` | runAsUser: 0, allowPrivilegeEscalation | C-0044 |
| `wildcard-clusterrole` | RBAC `*/*/*` | C-0034 |

---

## Etap 1 — Trivy Operator

Trivy Operator to pierwsza linia obrony — skanuje wszystko co jest w klastrze zanim cokolwiek złego się wydarzy. Po zainstalowaniu automatycznie wykrywa każdy nowy workload i uruchamia skan. Wyniki lądują jako CRDs w klastrze, więc można je odpytywać przez `kubectl` jak każdy inny zasób.

Skonfigurowaliśmy go żeby skupiał się tylko na HIGH i CRITICAL — MEDIUM i LOW zostawiamy na później, żeby nie tonąć w szumie przy pierwszym przeglądzie.

### Instalacja

```bash
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system --create-namespace \
  --set trivy.ignoreUnfixed=true \
  --set "trivy.severity=HIGH\,CRITICAL" \
  --set operator.scanJobsConcurrentLimit=1
```

### Wyniki — ConfigAuditReports

```bash
kubectl get configauditreports -n default
kubectl get configauditreports -n default <nazwa> -o json | jq '.report.checks[]'
```

| Workload | HIGH | MEDIUM | LOW |
|----------|------|--------|-----|
| privileged-container | 3 | 0 | 0 |
| hostpath-mount | 4 | 0 | 0 |
| hardcoded-secrets | 3 | 0 | 0 |
| run-as-root | 2 | 0 | 0 |

### Top findings

| ID | Severity | Opis |
|----|----------|------|
| AVD-KSV-0017 | HIGH | Privileged container |
| AVD-KSV-0121 | HIGH | Disallowed volumes (HostPath) |
| AVD-KSV-0118 | HIGH | Default security context |
| AVD-KSV-0014 | HIGH | Root filesystem not read-only |

### Eksport do CSV

```bash
kubectl get configauditreports -A -o json \
  | jq -r '.items[] | .metadata.name as $name |
    .report.checks[] | select(.success==false) |
    [$name, .severity, .checkID, .title] | @csv' \
  > trivy-audit.csv
```

---

## Etap 2 — Kubescape Operator

Kubescape patrzy na klaster z innej perspektywy niż Trivy — zamiast CVE w obrazach, sprawdza czy konfiguracja klastra jest zgodna z frameworkami bezpieczeństwa takimi jak CIS Benchmark czy MITRE ATT&CK. To pozwala zobaczyć "duży obraz" — jaki procent zasobów spełnia wymagania i które konkretnie kontrolki są naruszone.

Operator instaluje się raz i skanuje cyklicznie. W labie ustawiliśmy skan o 2:00 w nocy, ale do testów uruchamialiśmy go ręcznie przez job.

### Instalacja

```bash
helm repo add kubescape https://kubescape.github.io/helm-charts/
helm install kubescape kubescape/kubescape-operator \
  --namespace kubescape --create-namespace \
  --set clusterName=$(kubectl config current-context) \
  --set kubescape.scanSchedule="0 2 * * *"
```

### Ręczne uruchomienie skanu

```bash
kubectl create job -n kubescape \
  --from=cronjob/kubescape-scheduler manual-scan-$(date +%s)
```

### Wyniki

**Compliance score: 80/100**

| Severity | Liczba findingów |
|----------|-----------------|
| Critical | 0 |
| High | 73 |
| Medium | 177 |

### Top HIGH findings

| Kontrolka | Opis | Nasze workloady |
|-----------|------|-----------------|
| C-0017 | Privileged container | `privileged-container` |
| C-0045 | HostPath mount | `hostpath-mount` |
| C-0012 | Misplaced secrets | `hardcoded-secrets` |
| C-0034 | Wildcard RBAC | `wildcard-clusterrole` |
| C-0009 | Brak CPU/memory limits | wszystkie |
| C-0044 | Brak security context | wszystkie |

---

## Etap 3 — Falco

Trivy i Kubescape działają statycznie — mówią co jest źle skonfigurowane. Falco wchodzi w grę gdy klaster już działa i trzeba łapać podejrzane zachowania w czasie rzeczywistym. Działa przez eBPF: podpina się do kernela i obserwuje syscalle ze wszystkich kontenerów bez konieczności modyfikowania obrazów.

Falco sam w sobie tylko wykrywa i alarmuje — nie blokuje. To jego zaleta przy wdrożeniu: można włączyć w trybie "obserwacji" bez ryzyka że coś przerwie działanie produkcji.

Do testów użyliśmy Falcosidekick — komponent który odbiera alerty od Falco i rozsyła je dalej (Slack, Splunk, Prometheus, własne webhooki). W labie włączyliśmy jego wbudowane WebUI żeby zobaczyć alerty w przeglądarce.

Napotkaliśmy jeden problem: domyślny driver eBPF (legacy) nie miał prebuilt binarki dla kernela GKE 6.12.55+. Fix był prosty — przełączenie na `modern_ebpf` który jest wbudowany bezpośrednio w kernel i nie wymaga żadnego zewnętrznego drivera.

### Instalacja

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falco.json_output=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true \
  --set driver.kind=modern_ebpf
```

> **Uwaga:** `driver.kind=ebpf` (legacy) nie działało na GKE — brak prebuilt drivera dla kernela 6.12.55+.
> Użyto `modern_ebpf` który nie wymaga zewnętrznego drivera.

### Falcosidekick WebUI

```bash
kubectl port-forward -n falco svc/falco-falcosidekick-ui 2802:2802
# http://localhost:2802 — login: admin / hasło: admin
```

### Wygenerowane eventy testowe

```bash
# Odczyt /etc/shadow
kubectl exec -n default <pod> -- cat /etc/shadow

# Szukanie kluczy prywatnych
kubectl exec -n default <pod> -- find / -name 'id_rsa' 2>/dev/null

# Dostęp do hosta przez HostPath
kubectl exec -n default <hostpath-pod> -- cat /host/etc/shadow
```

### Wykryte alerty

| Severity | Reguła | Trigger |
|----------|--------|---------|
| Warning | Read sensitive file untrusted | `cat /etc/shadow` |
| Warning | Search Private Keys or Passwords | `find / -name id_rsa` |
| Notice | Contact K8S API Server From Container | Kubescape node-agent |
| Notice | Packet socket created in container | Kubescape node-agent |

### Podgląd alertów w terminalu

```bash
kubectl logs -n falco <falco-pod> -c falco \
  | grep "^{" \
  | jq -r '"\(.priority) | \(.rule) | \(.output_fields["k8s.pod.name"])"' \
  | sort | uniq -c | sort -rn
```

---

## Etap 4 — Tetragon

Tetragon idzie krok dalej niż Falco: zamiast tylko alarmować, potrafi aktywnie blokować. Też działa przez eBPF i też ma minimalny overhead, ale daje możliwość wysłania SIGKILL do procesu zanim cokolwiek złego się stanie — np. zanim shell w produkcyjnym podzie zdąży cokolwiek wykonać.

Polityki definiuje się jako `TracingPolicy` — można targetować konkretne syscalle, filtrować po namespace, nazwie poda, labelu. W labie napisaliśmy politykę która blokuje uruchomienie `/bin/bash` i `/bin/sh` w podzie `privileged-container`, ale pozwala na to w pozostałych podach.

Tetragon miał poważniejszy problem z instalacją niż Falco: crashował na domyślnym nodzie GKE (COS) z błędem BTF dla modułu kernela `nls_cp437`. To znany bug który nie był jeszcze naprawiony w żadnej z dostępnych wersji (1.4–1.6). Jedynym obejściem okazało się użycie node poola z Ubuntu zamiast COS — tam kernel ma poprawne BTF i Tetragon startuje bez problemu.

### Instalacja

```bash
helm repo add cilium https://helm.cilium.io
helm install tetragon cilium/tetragon \
  --namespace kube-system
```

> **Uwaga:** Tetragon crashował na COS node (domyślny GKE image) z błędem:
> `load BTF for kmod nls_cp437: read string section: string table is empty`
> **Fix:** Użycie node pool z `UBUNTU_CONTAINERD` image.

### TracingPolicy — blokowanie shell

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-shell-in-default
spec:
  podSelector:
    matchLabels:
      app: privileged-container
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
      matchActions:
      - action: Sigkill
```

```bash
kubectl apply -f block-shell-policy.yaml
```

### Wyniki testu

```bash
# ZABLOKOWANE (exit 137 = SIGKILL)
kubectl exec -n default privileged-container-xxx -- /bin/bash -c "echo test"
# → command terminated with exit code 137

# DOZWOLONE (inny pod, brak matchLabels)
kubectl exec -n default hardcoded-secrets-xxx -- /bin/bash -c "echo test"
# → test (exit 0)
```

---

## Napotkane problemy

### Narzędzia są zasobożerne — bardziej niż się wydaje

Każde z czterech narzędzi działa jako osobny operator lub DaemonSet i definiuje własne `resource requests`. Gdy uruchomimy je wszystkie na raz, suma tych requestów szybko przekracza to co oferuje mały node — scheduler K8s zaczyna zostawiać nowe pody w stanie `Pending`, bo nie ma gdzie ich umieścić. Nie chodzi o faktyczne zużycie CPU czy RAM w danej chwili, ale o zarezerwowaną pojemność.

W praktyce: na pojedynczym nodzie `e2-standard-2` (2 vCPU, 8 GB) cztery operatory mieszczą się bez problemu. Mniejsze instancje (e2-small, e2-medium) okazały się niewystarczające. Przy planowaniu produkcyjnego wdrożenia warto z góry zarezerwować dedykowane nody dla narzędzi monitorujących i nie mieszać ich z workloadami aplikacyjnymi.

### Tetragon wymaga Ubuntu — nie działa na domyślnym obrazie GKE

GKE domyślnie używa Container-Optimized OS (COS) jako obrazu nodów. Tetragon na tym systemie nie uruchamia się poprawnie ze względu na problem z metadanymi BTF jednego z modułów kernela (`nls_cp437`). Jest to znany bug obecny w Tetragon 1.4–1.6, niezależny od wersji GKE. Obejście: node pool z obrazem `UBUNTU_CONTAINERD` — na Ubuntu kernel ma kompletne BTF i Tetragon startuje bez problemów.

### Falco wymaga nowszego sterownika eBPF na GKE

Domyślny driver eBPF Falco (legacy) wymaga prebuilt binarki skompilowanej pod konkretną wersję kernela. GKE regularnie aktualizuje kernele i bywa że dla najnowszych wersji prebuilt binarki jeszcze nie ma. Rozwiązanie: użycie `modern_ebpf` — sterownika wbudowanego bezpośrednio w kernel (dostępnego od Linux 5.13+), który nie wymaga żadnych zewnętrznych plików i działa na wszystkich aktualnych wersjach GKE.

---

## Wnioski

### Co działało dobrze

Wszystkie cztery narzędzia wykryły to co miały wykryć. Celowo złe konfiguracje — privileged container, hostpath mount, hardcoded secrets, wildcard RBAC — pojawiły się w wynikach Trivy i Kubescape bez żadnej dodatkowej konfiguracji. Falco od razu zaczął łapać podejrzane syscalle po wdrożeniu. Tetragon po wgraniu `TracingPolicy` skutecznie blokował shell exit 137, podczas gdy inne pody działały normalnie.

Szczególnie warte uwagi: **Falco wychwycił aktywność Kubescape node-agenta** (połączenia do K8s API Server) jako podejrzaną. To pokazuje że narzędzia do security potrafią wzajemnie na siebie alarmować — w produkcji trzeba skonfigurować białe listy dla własnych operatorów.

### Różnice między narzędziami

Trivy i Kubescape pokrywają podobny obszar (statyczna analiza), ale z różnych kątów. Trivy jest bardziej szczegółowy na poziomie pojedynczego workloadu i CVE. Kubescape daje lepszy obraz "compliance całego klastra" i mapowanie na frameworki (CIS, MITRE) — przydatne przy audytach.

Falco i Tetragon też się uzupełniają: Falco jest lepszy do szerokiej detekcji i integracji z zewnętrznymi systemami (Splunk, Slack), Tetragon do precyzyjnego enforcement na poziomie konkretnych procesów i namespaces.

### Co sprawiło trudność

Główne wyzwania były infrastrukturalne, nie związane z samymi narzędziami:
- **Zasoby nodów** — uruchomienie czterech operatorów jednocześnie na małych nodach (e2-small) nie działa. W praktyce każdy operator ma swoje resource requests i trzeba to uwzględnić przy planowaniu klastra.
- **Kompatybilność kernela** — nowsze kernele GKE (6.12+) mają niespójności w BTF dla niektórych modułów, co blokuje Tetragon na COS. Ubuntu node pool rozwiązuje problem, ale to coś do śledzenia przy upgrade'ach GKE.
- **Sekwencja instalacji** — instalowanie wszystkiego naraz powoduje resource contention. W produkcji warto wdrażać narzędzia etapami.

### Dalsze kroki

To co przetestowaliśmy to punkt startowy. W środowisku produkcyjnym warto dodać:
- Centralna agregacja wyników z wielu klastrów (webhook + PostgreSQL lub ARMO Platform)
- Integracja Falcosidekick → Splunk HEC dla runtime eventów
- Tuning reguł Falco — domyślne reguły generują dużo szumu od systemowych operatorów
- Tetragon w trybie audit przed włączeniem enforcement — żeby zobaczyć co by blokował zanim faktycznie zacznie

---

## Cleanup — usunięcie klastra

```bash
gcloud container clusters delete kubescape-vulnerable-lab \
  --zone=europe-west1-b
```

> Koszt klastra: ~$0.07/h (e2-standard-2, 1 node Ubuntu).
