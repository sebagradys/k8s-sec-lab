# K8s Security Lab

Open source K8s security aggregation — alternatywa dla komercyjnych CNAPPów dla firm z dużymi środowiskami hybrid cloud. Projekt dokumentuje architekturę, koszty wdrożenia i ROI dla organizacji które chcą centralnie agregować podatności CVE i eventy runtime z setek klastrów Kubernetes bez vendor lock-in i bez 6-cyfrowych licencji rocznych.

## Raporty

- [Lab Report](https://sebagradys.github.io/k8s-sec-lab/lab-report.html) — przebieg labu, wyniki skanów, napotkane problemy, wnioski
- [Architektura — Azure Streaming](https://sebagradys.github.io/k8s-sec-lab/architecture-azure.html) — diagram architektury multi-cloud: Trivy + Falco → Event Hubs → Databricks → ADLS Gen2 → Synapse → PowerBI
- [Raport kosztowy — Azure Streaming](https://sebagradys.github.io/k8s-sec-lab/cost-report.html) — TCO dla przykładowego środowiska hybrydowego, scenariusz 5× CVE, private deployment
- [Wariant Lean — CVE Only](https://sebagradys.github.io/k8s-sec-lab/cve-lean.html) — architektura i koszty dla samej agregacji podatności (Trivy → ADLS → Databricks → PowerBI), wariant uproszczony
