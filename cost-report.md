# Raport kosztowy — K8s Security Monitoring (Azure Streaming Architecture)

> Trivy + Falco na klastrach · Azure Event Hubs · Databricks · ADLS Gen2 · Synapse · PowerBI
> Środowisko: multi-cloud (AKS · GKE · on-prem) · scenariusz 5× wolumen CVE · private deployment

---

## Środowisko

| Typ | Nody | Pokrycie |
|-----|------|----------|
| On-Premises | 3 500 | CVE (wszystkie) + Runtime (tylko prod) |
| AKS (Azure) | 400 | CVE (wszystkie) + Runtime (tylko prod) |
| GKE (GCP) | 50 | CVE (wszystkie) + Runtime (tylko prod) |
| **Razem** | **3 950** | |
| Prod (50%) | ~1 975 nodów | Trivy + Falco |
| Dev/Test (50%) | ~1 975 nodów | tylko Trivy |

Szacunkowa liczba klastrów: ~79 (przy średniej 50 nodów/klaster).
Szacunkowa liczba podów: ~31 600 (przy średniej 8 podów/node).

---

## Wolumen danych do Event Hubs

Przyjęte założenia:
- Trivy skanuje przy zmianie obrazu (~10% podów/dzień) + tygodniowy pełny rescan
- Scenariusz **5× CVE** (częstsze skany, większe środowisko)
- Falco **po tuningu** — ~200 alertów/h per klaster prod
- ~25 klastrów prod aktywnie monitorowanych runtime

| Pipeline | Wolumen/miesiąc |
|----------|----------------|
| CVE events (Trivy, wszystkie nody) | 60–90 GB |
| Runtime events (Falco, prod only) | ~4 GB |
| Audit logi (Fluent Bit, prod only) | ~18 GB |
| **Łącznie do Event Hubs** | **~115 GB/miesiąc** |

---

## Koszty Azure — miesięcznie

### Standard vs Private/Premium

| Komponent | Standard | Private / Premium | Co daje Premium |
|-----------|---------|-------------------|-----------------|
| **Azure Event Hubs** | $230 | **$520** | Dedykowane PUs, private endpoint, brak shared infra, CMK, 99.99% SLA |
| **Azure Databricks** | $350 | **$680** | VNet injection, private link, Unity Catalog, Azure AD SSO, audit logs |
| **ADLS Gen2** | $50 | **$90** | Private endpoint, Customer-Managed Keys, RA-GRS geo-redundancy |
| **Azure Synapse Serverless** | $20 | **$50** | Managed VNet, private endpoints, CMK, Purview integration |
| **Azure Stream Analytics** | $60 | **$60** | ¹ |
| **PowerBI Premium Per User** | $150 (Pro, 15u) | **$600** (PPU, 30u) | Paginated reports, deployment pipelines, XMLA endpoint |
| **Key Vault + Private DNS + Monitor** | — | **$35** | CMK dla wszystkich usług, private DNS resolution |
| **Łącznie miesięcznie** | **~$860** | **~$2 035** | |
| **Łącznie rocznie** | **~$10 300** | **~$24 400** | |

> ¹ Stream Analytics w private deployment wymagałby dedykowanego klastra (~$800/m). Zamiast tego używamy Databricks Structured Streaming (już w budżecie). Stream Analytics standard pozostaje jako router alertów CRITICAL → SIEM.

**Delta za private deployment: +$1 175/miesiąc (~+$14 100/rok)**
Główne składowe: Event Hubs Premium (+$290), PowerBI PPU (+$450), Databricks Premium (+$330).

---

## Koszty wdrożenia — Man-Days

### Faza 1 — Infrastruktura Azure (IaC, networking, IAM)

| Zadanie | MD |
|---------|----|
| Terraform modules: Event Hubs, ADLS, Databricks, Synapse, Key Vault | 20 |
| VNet, private endpoints, private DNS zones (6+ usług) | 12 |
| RBAC design + Azure AD groups + service principals | 8 |
| Customer-Managed Keys (Key Vault + rotation policy) | 5 |
| CI/CD pipeline (Azure DevOps) dla IaC | 8 |
| Środowiska dev/staging/prod | 15 |
| **Suma** | **68 MD** |

### Faza 2 — Warstwa klastrów (Trivy + Falco + producenci)

| Zadanie | MD |
|---------|----|
| Helm charts: Trivy Operator + konfiguracja per środowisko | 8 |
| Helm charts: Falco + Falcosidekick (output Kafka → Event Hubs) | 8 |
| Falco rules tuning — allowlist, eliminacja szumu | 20 |
| Kafka Producer CronJob (VulnReports → Event Hubs) | 6 |
| Fluent Bit (audit logi → Event Hubs) | 5 |
| GitOps setup (ArgoCD/Flux) + application sets per klaster | 10 |
| Pilot deployment (1× AKS, 1× GKE, 1× on-prem) + walidacja | 12 |
| Staged rollout plan + dokumentacja operacyjna | 8 |
| **Suma** | **77 MD** |

### Faza 3 — Data Pipeline (Databricks + Stream Analytics)

| Zadanie | MD |
|---------|----|
| Schema design: Bronze/Silver/Gold dla CVE, runtime, audit | 8 |
| Unity Catalog setup: namespaces, access control, data lineage | 6 |
| CVE Enrichment Job: deduplikacja, normalizacja, join CMDB | 18 |
| CVE Aggregation Job: Bronze→Silver→Gold, compliance scores | 15 |
| Audit Analysis Job: join VulnReport + audit log | 10 |
| Stream Analytics job: CRITICAL filter + allowlist → sec.alerts.critical | 8 |
| SIEM connector (sec.alerts.critical → Splunk/QRadar) | 8 |
| Delta Lake optimization: Z-ordering, bloom filters | 6 |
| Error handling, dead-letter queues, retry logic | 8 |
| Unit testy Spark jobs | 12 |
| Integration testy end-to-end | 12 |
| Performance testy (symulacja 5× wolumenu CVE) | 8 |
| **Suma** | **119 MD** |

### Faza 4 — Reporting i dashboardy

| Zadanie | MD |
|---------|----|
| Synapse Serverless SQL views na Gold layer | 8 |
| PowerBI data model design | 6 |
| PowerBI dashboardy (CVE heatmapa, compliance score, trend, Falco alert rate) | 18 |
| Row-Level Security: team owner widzi tylko swoje klastry | 6 |
| Synapse Pipeline: tygodniowy CSV/Excel export | 6 |
| Logic App: e-mail/Teams dystrybucja raportu | 4 |
| UAT z security teamem + iteracje | 10 |
| **Suma** | **58 MD** |

### Faza 5 — Testy, bezpieczeństwo, go-live

| Zadanie | MD |
|---------|----|
| Security review: IAM audit, data flow, private endpoints | 10 |
| Disaster recovery test: Event Hubs failover, ADLS restore | 8 |
| Load test: wszystkie klastry wysyłają jednocześnie | 8 |
| Azure Monitor alerting dla health pipeline'u | 6 |
| Runbooks operacyjne (upgrade Falco, rotacja kluczy, skalowanie) | 10 |
| Handover + szkolenie ops teamu | 8 |
| Project management (cały czas trwania projektu) | 20 |
| **Suma** | **70 MD** |

### Łącznie

| Faza | MD |
|------|----|
| 1 — Infrastruktura Azure | 68 |
| 2 — Klastry (Trivy + Falco) | 77 |
| 3 — Data Pipeline | 119 |
| 4 — Reporting | 58 |
| 5 — Testy + go-live | 70 |
| **TOTAL (base)** | **392 MD** |
| **TOTAL z buforem 15%** | **~450 MD** |

---

## Skład zespołu

| Rola | Liczba osób | Główne fazy | MD |
|------|------------|-------------|-----|
| Cloud / Platform Engineer | 2 | 1, 5 | 80 |
| Data Engineer | 3 | 1, 3, 4 | 135 |
| K8s / DevOps Engineer | 2 | 2, 5 | 90 |
| Security Engineer | 1 | 2, 5 | 45 |
| BI Developer | 1 | 4 | 40 |
| QA / Performance Engineer | 1 | 3, 5 | 40 |
| Project Manager | 1 | all | 20 |
| **Razem** | **11 osób** | | **~450 MD** |

Peak równoległy (~8 osób) przypada na Fazę 3.

---

## Timeline

```
Miesiąc:   1     2     3     4     5     6     7     8     9
           ├─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
Faza 1:    ████████████
Faza 2:          ████████████████
Faza 3:                ████████████████████
Faza 4:                            ██████████
Faza 5:                                  ████████████████
```

**Realistyczny czas: 7–9 miesięcy** od kickoffu do produkcyjnego go-live (rolling deployment na wszystkich klastrach).

---

## Łączny koszt projektu (TCO rok 1)

| Składnik | Standard | Private / Premium |
|----------|---------|-------------------|
| Wdrożenie (450 MD × 2 000 PLN) | 900 000 PLN | 900 000 PLN |
| Azure — rok 1 | ~47 000 PLN (~$10 300) | ~112 000 PLN (~$24 400) |
| **TCO Rok 1** | **~947 000 PLN** | **~1 012 000 PLN** |
| **TCO Rok 2+** (tylko Azure) | **~47 000 PLN/rok** | **~112 000 PLN/rok** |

### Porównanie z komercyjnym CNAPP

Wiz / Prisma Cloud na 3 950 nodach: szacunkowo **$400 000–800 000/rok** (licencja).
Open source + Azure backend break-even: **~rok 3–4** przy wariancie standard, **~rok 5–6** przy private/premium.

---

## Czynniki ryzyka kosztowego

| Ryzyko | Wpływ | Mitygacja |
|--------|-------|-----------|
| Falco bez tuningu — 10–100× więcej eventów | +$200–600/m Event Hubs + Databricks | Dedykowany czas w Fazie 2 (20 MD) na tuning |
| Pełny dzienny rescan wszystkich obrazów Trivy | +$200–400/m | Konfiguracja Trivy: skan tylko przy zmianie + tygodniowy full |
| PowerBI Premium P1 zamiast PPU | +$4 400/m | PPU wystarczy dla ≤300 użytkowników |
| Databricks all-purpose clusters (nie jobs) | ×3–5 kosztu compute | Wymuszenie job clusters z auto-termination |
| Rollout na 79 klastrów z problemami kompatybilności | +20–40 MD | Pilot na 3 klastrach (1 per typ) przed masowym rollout |
