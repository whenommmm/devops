# Pre-written sessions (for when Claude isn't available)

These follow your schedule in `STUDY_PLAN.md`. Do them **in order**, 2 per day.

| # | File | Day | Topic |
|---|------|-----|-------|
| 1 | `01_git-github.md` | Oct 11 · S1 | Git & GitHub |
| 2 | `02_docker-fundamentals.md` | Oct 11 · S2 | Docker I: containers, architecture, images, lifecycle |
| 3 | `03_dockerfile.md` | Oct 12 · S1 | Docker II: Dockerfile |
| 4 | `04_docker-networking-volumes-compose.md` | Oct 12 · S2 | Docker III: volumes, networking, YAML, Compose |
| 5 | `05_kubernetes-fundamentals.md` | Oct 13 · S1 | Kubernetes I: why, architecture, kubectl, manifests |
| 6 | `06_pods-replicasets-deployments.md` | Oct 13 · S2 | Kubernetes II: Pods, labels, ReplicaSets, Deployments |
| 7 | `07_kubernetes-networking-services.md` | Oct 14 · S1 | Kubernetes III: Pod networking, Services, DNS |
| 8 | `08_ingress-configmaps-secrets.md` | Oct 14 · S2 | Kubernetes IV: Ingress, ConfigMaps, Secrets |
| 9 | `09_storage-probes-hpa.md` | Oct 15 · S1 | Kubernetes V: storage, probes, HPA |
| 10 | `10_kubernetes-troubleshooting.md` | Oct 15 · S2 | Kubernetes VI: troubleshooting |
| 11 | `11_helm.md` | Oct 16 · S1 | Helm |
| 12 | `12_cicd-github-actions.md` | Oct 16 · S2 | CI/CD & GitHub Actions |
| 13 | `13_complete-cicd-devsecops.md` | Oct 17 · S1 | Complete CI/CD & DevSecOps |
| 14 | `14_monitoring-observability-gitops.md` | Oct 17 · S2 | Monitoring, Observability & GitOps |
| 15 | `15_iac-terraform-cloud.md` | Oct 18 · S1 | IaC, Terraform & Cloud |
| 16 | `16_final-project-mock.md` | Oct 18 · S2 | Final project walkthrough + mini mock exam |

**Extra reading:** `../notes/extras_foundations.md` (small-but-important topics from Intro, Linux, Shell, Networking, Git) and `../notes/GLOSSARY.md` (all full forms).

## How to use each session

1. **Read one concept at a time.** Do the 🛠 Mac task in that concept.
2. **Answer the ✅ checks** in `MY_ANSWERS.md` (the template is ready) **before** opening the answer key.
3. At the end, do the **📝 session quiz** (4 MCQs + 2 one-liners), also in `MY_ANSWERS.md`.
4. **Then** open the matching `_ANSWERS.md` file and mark yourself. Each answer repeats the question, all options, the correct one, and why.
5. Mark guesses with `?`. Write "idk" rather than leaving blanks.

When Claude is back, say **"grade my answers"**. Claude will read `MY_ANSWERS.md`, update your tracker and weak-topic list, and re-test anything you missed.

## Your tools (already installed and working)

You're using **Colima** instead of Docker Desktop. It runs Docker inside a small Ubuntu Linux virtual machine. The `docker` and `docker compose` commands work exactly the same.
- Check it's running: `colima status`. If it isn't: `colima start`.
- Kubernetes runs in **minikube**. Check: `minikube status`. If it isn't running: `minikube start`. Then `kubectl get nodes` should show `minikube   Ready`.
- Ignore any "Docker Desktop" instructions in older notes. Use Colima/minikube.
