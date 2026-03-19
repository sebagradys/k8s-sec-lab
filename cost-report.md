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

Szacunkowa liczba klastrów: **455** (365 on-prem + 90 cloud AKS/GKE).
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

> **Faza 1 jest już zrealizowana** — infrastruktura Azure (IaC, networking, IAM, Key Vault, środowiska) została wdrożona. Poniższe zestawienie obejmuje wyłącznie pozostałe fazy (2–5). Faza 1 pokazana dla referencji jako zakończona.

### ~~Faza 1 — Infrastruktura Azure (IaC, networking, IAM)~~ ✅ ZREALIZOWANA

| Zadanie | MD |
|---------|----|
| Terraform modules: Event Hubs, ADLS, Databricks, Synapse, Key Vault | 20 |
| VNet, private endpoints, private DNS zones (6+ usług) | 12 |
| RBAC design + Azure AD groups + service principals | 8 |
| Customer-Managed Keys (Key Vault + rotation policy) | 5 |
| CI/CD pipeline (Azure DevOps) dla IaC | 8 |
| Środowiska dev/staging/prod | 15 |
| ~~**Suma**~~ | ~~**68 MD**~~ — **nie wliczana w koszt** |

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
| CVE Enrichment Job: deduplikacja, normalizacja, join CMDB | 12 |
| CVE Aggregation Job: Bronze→Silver→Gold, compliance scores | 9 |
| Audit Analysis Job: join VulnReport + audit log | 7 |
| Stream Analytics job: CRITICAL filter + allowlist → sec.alerts.critical | 8 |
| SIEM connector (sec.alerts.critical → Splunk/QRadar) | 8 |
| Delta Lake optimization: Z-ordering, bloom filters | 6 |
| Error handling, dead-letter queues, retry logic | 8 |
| Integration testy end-to-end | 12 |
| **Suma** | **84 MD** |

### Faza 4 — Reporting i dashboardy

| Zadanie | MD |
|---------|----|
| Synapse Serverless SQL views na Gold layer | 4 |
| PowerBI data model design | 3 |
| PowerBI dashboardy (CVE heatmapa, compliance score, trend, Falco alert rate) | 9 |
| Row-Level Security: team owner widzi tylko swoje klastry | 3 |
| Synapse Pipeline: tygodniowy CSV/Excel export | 4 |
| UAT z security teamem + iteracje | 5 |
| **Suma** | **28 MD** |

### Faza 5 — Testy, bezpieczeństwo, go-live

| Zadanie | MD |
|---------|----|
| Security review: IAM audit, data flow, private endpoints | 10 |
| Disaster recovery test: Event Hubs failover, ADLS restore | 8 |
| Load test: wszystkie klastry wysyłają jednocześnie | 8 |
| Azure Monitor alerting dla health pipeline'u | 6 |
| Runbooks operacyjne (upgrade Falco, rotacja kluczy, skalowanie) | 10 |
| Handover + szkolenie ops teamu | 8 |
| **Suma** | **50 MD** |

### Łącznie

| Faza | MD | Status |
|------|----|--------|
| ~~1 — Infrastruktura Azure~~ | ~~68~~ | ✅ Zrealizowana |
| 2 — Klastry (Trivy + Falco) | 77 | do realizacji |
| 3 — Data Pipeline | 84 | do realizacji |
| 4 — Reporting | 28 | do realizacji |
| 5 — Testy + go-live | 50 | do realizacji |
| **TOTAL pozostałe (base)** | **239 MD** | |
| **TOTAL z buforem 15%** | **~275 MD** | |

---

## Skład zespołu (Fazy 2–5)

| Rola | Liczba osób | Główne fazy | MD |
|------|------------|-------------|-----|
| ~~Cloud / Platform Engineer~~ | ~~2~~ | ~~1~~, 5 | ~~80~~ → 15 (tylko Faza 5) |
| Data Engineer | 2 | 3, 4 | 80 |
| K8s / DevOps Engineer | 2 | 2, 5 | 80 |
| Security Engineer | 1 | 2, 5 | 40 |
| BI Developer | 1 | 4 | 20 |
| QA / Performance Engineer | 1 | 3, 5 | 40 |
| **Razem** | **8 osób** | Fazy 2–5 | **~275 MD** |

Peak równoległy (~6 osób) przypada na Fazę 3. Cloud/Platform Engineer zaangażowany jedynie w Fazę 5 (DR test, runbooks).

---

## Timeline (od teraz — Fazy 2–5)

```
Miesiąc:   1     2     3     4     5     6     7
           ├─────┼─────┼─────┼─────┼─────┼─────┤
Faza 1:    [✅ DONE]
Faza 2:    ████████████████
Faza 3:          ████████████████████
Faza 4:                      ██████████
Faza 5:                            ████████████
```

**Realistyczny czas od teraz: 5–7 miesięcy** do produkcyjnego go-live (rolling deployment na wszystkich klastrach).

---

## Łączny koszt projektu (TCO)

> Faza 1 zrealizowana — wdrożenie obejmuje wyłącznie Fazy 2–5 (~275 MD).
> Kursy NBP: 1 USD = 3,72 PLN · 1 EUR = 4,28 PLN.

| Składnik | Standard | Private / Premium |
|----------|---------|-------------------|
| Wdrożenie Fazy 2–5 (275 MD × 3 200 PLN) | 880 000 PLN (~205 600 EUR) | 880 000 PLN (~205 600 EUR) |
| Azure — rok 1 | ~38 300 PLN (~$10 300) | ~90 800 PLN (~$24 400) |
| **TCO Rok 1** | **~918 000 PLN** (~214 500 EUR) | **~971 000 PLN** (~227 000 EUR) |
| **TCO Rok 2+** (tylko Azure) | **~38 300 PLN/rok** (~9 000 EUR) | **~90 800 PLN/rok** (~21 200 EUR) |

### Porównanie z benchmarkiem rynkowym

| | Koszt rok 1 | Koszt rok 2+ | Uwagi |
|--|------------|-------------|-------|
| **Nasze rozwiązanie (Private/Premium)** | ~227 000 EUR | ~21 200 EUR/rok | Fazy 2–5 + Azure |
| **Zewnętrzny integrator + Azure** | ~221 200 EUR | ~21 200 EUR/rok | 200 000 EUR integrator + Azure |
| **Komercyjny CNAPP (benchmark)** | ~667 000 EUR | ~667 000 EUR/rok | licencja Wiz/Prisma Cloud |

**Oszczędność względem CNAPPa:**
- Rok 1: ~440 000 EUR
- Rok 2: ~646 000 EUR
- Rok 3+: ~646 000 EUR/rok

Rozwiązanie zwraca się względem CNAPPa **natychmiast — już w roku pierwszym**. Koszt wdrożenia (własny lub przez integratora) jest porównywalny i niższy niż jedna rata roczna licencji komercyjnej.

---

## Czynniki ryzyka kosztowego

| Ryzyko | Wpływ | Mitygacja |
|--------|-------|-----------|
| Falco bez tuningu — 10–100× więcej eventów | +$200–600/m Event Hubs + Databricks | Dedykowany czas w Fazie 2 (20 MD) na tuning |
| Pełny dzienny rescan wszystkich obrazów Trivy | +$200–400/m | Konfiguracja Trivy: skan tylko przy zmianie + tygodniowy full |
| PowerBI Premium P1 zamiast PPU | +$4 400/m | PPU wystarczy dla ≤300 użytkowników |
| Databricks all-purpose clusters (nie jobs) | ×3–5 kosztu compute | Wymuszenie job clusters z auto-termination |
| Rollout na 455 klastrów z problemami kompatybilności | +20–40 MD | Pilot na 3 klastrach (1 per typ) przed masowym rollout |
