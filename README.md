# K8s Security Lab

Praktyczny lab z monitorowania bezpieczeństwa Kubernetes — Trivy, Kubescape, Falco, Tetragon na GKE.

## Raporty

- [Lab Report](https://sebagradys.github.io/k8s-sec-lab/lab-report.html) — przebieg labu, wyniki skanów, napotkane problemy, wnioski
- [Architektura — Azure Streaming](https://sebagradys.github.io/k8s-sec-lab/architecture-azure.html) — diagram architektury multi-cloud: Trivy + Falco → Event Hubs → Databricks → ADLS Gen2 → Synapse → PowerBI
- [Raport kosztowy — Azure Streaming](https://sebagradys.github.io/k8s-sec-lab/cost-report.html) — TCO dla 3 950 nodów, scenariusz 5× CVE, private deployment, 275 MD wdrożenia
- [Wariant Lean — CVE Only](https://sebagradys.github.io/k8s-sec-lab/cve-lean.html) — architektura i koszty dla samej agregacji podatności (Trivy → ADLS → PowerBI), ~85 MD, ~65k EUR rok 1
