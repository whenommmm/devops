# Labs · Session 1 · DevOps Engineer Roadmap

## Lab 1 · Draw a CI/CD workflow from code commit to deployment (build, test, package, deploy, monitor)
**Scenario used:** an online shop web app kept in a Git repo on GitHub.
CI = Continuous Integration (every push is built and tested automatically). CD = Continuous Delivery/Deployment (every tested build is automatically made ready for release / released).

**Answer (what to draw in the exam):**
```text
 Developer
    │  git commit + git push  (or opens a Pull Request)
    ▼
 [0 SOURCE]   Git repo (GitHub) ── push event triggers the pipeline
    │
    ▼
 [1 BUILD]    install dependencies, compile the code          ┐
    │                                                          │  CI
    ▼                                                          │
 [2 TEST]     unit tests + lint + security scan                │
    │         any failure → pipeline STOPS, developer notified ┘
    ▼
 [3 PACKAGE]  docker build → tag image shop:1.4.0 → push to registry
    │
    ▼
 [4 DEPLOY]   deploy to staging → smoke test → (approval) → production   CD
    │
    ▼
 [5 MONITOR]  metrics, logs, alerts (Prometheus + Grafana)
    │
    └──── feedback (bugs, alerts, slow pages) ────▶ back to Developer
```
**What each stage does:**
| Stage | What happens | Example tool | Output |
|---|---|---|---|
| Build | fetch code, install dependencies, compile | GitHub Actions / Jenkins | a build that compiles |
| Test | run automated tests and code checks | pytest / Jest, SonarQube | pass or fail report |
| Package | wrap the app + its dependencies into one versioned artifact | Docker, image registry | `shop:1.4.0` image |
| Deploy | release the same artifact to staging, then production | kubectl / Helm / Argo CD | app live for users |
| Monitor | watch health, errors, response time; alert on problems | Prometheus, Grafana | dashboards + alerts |

- **Fail fast:** a failed stage stops the pipeline, so broken code never reaches users.
- **Build once, deploy many:** the same image goes to staging and to production.
- **Continuous Delivery** = a human clicks "approve" before production; **Continuous Deployment** = production release is fully automatic.

**🧠 In simple words:** Every time a developer pushes code, a pipeline automatically builds it, tests it, packs it into a container image, deploys it, and then watches it in production. If any step fails, the pipeline stops and tells the developer, and monitoring feeds problems back so the next change fixes them. In short: commit → build → test → package → deploy → monitor → feedback, all automatic.

## Lab 2 · Monolith vs microservices for an online shop: 2 operational benefits + 2 challenges of microservices
**Answer (what to write in the exam):**
```text
 MONOLITH                                MICROSERVICES
 ┌──────────────────────────────┐              [ API Gateway ]
 │ ONE app, ONE deploy           │       ┌──────────┬─────┴────┬───────────┐
 │ UI | Catalog | Cart |         │    Catalog     Cart      Orders     Payment
 │ Orders | Payment              │    service    service    service    service
 └──────────────┬───────────────┘      │          │          │           │
          one shared database         DB         DB         DB          DB
```
| Point | Monolith | Microservices |
|---|---|---|
| Code & deploy | one codebase, one deploy unit | many small services, each deployed on its own |
| Scaling | copy the whole app | scale only the busy service (e.g. Catalog) |
| Failure | one bug can crash the whole shop | failure stays mostly inside one service |
| Tech stack | one language/framework | each service can choose its own |
| Ops effort | simple: one app, one log, one pipeline | complex: many pipelines, network calls, logs |
| Best for | small team, early product | big team, high traffic, fast-changing features |

**2 operational benefits of microservices**
1. **Independent deployment:** the Payment team can release a fix without redeploying Catalog or Cart, so releases are smaller, faster and less risky.
2. **Independent scaling + fault isolation:** on a sale day you scale only Catalog/Cart; if Payment crashes, customers can still browse and add to cart.

**2 challenges of microservices**
1. **Operational complexity:** many services means many pipelines, containers, configs and service-to-service network calls (needs Docker, Kubernetes, service discovery, retries/timeouts).
2. **Monitoring and debugging are harder:** one customer order passes through 4 services, so you need centralised logging, metrics and distributed tracing to find which one failed. (Also: keeping data consistent across separate databases, e.g. order saved but payment failed.)

**🧠 In simple words:** A monolith is the whole shop in one program, easy to start but every change redeploys everything and one bug can take down the whole site. Microservices split the shop into small services (catalog, cart, orders, payment) that deploy and scale separately, at the cost of much more infrastructure and harder troubleshooting. In short: microservices buy independent deploys and scaling, and you pay for it in operational complexity.

## Lab 3 · One-page DevOps roadmap for a small team that deploys manually
**Answer (what to write in the exam):**
```text
 TODAY: code on laptops → copy files to server by hand (FTP/SSH) → hope it works
 GOAL : push code → tested automatically → deployed automatically → monitored

 Phase 1 ──▶ Phase 2 ──▶ Phase 3 ──▶ Phase 4 ──▶ Phase 5
 Version     Automated   Container-  CI/CD       Monitoring
 control     tests       ization     pipeline    + alerts
 (wk 1-2)    (wk 3-4)    (wk 5-6)    (wk 7-8)    (wk 9-10)
```
| # | Practice | What to do | Tool | Why this comes at this position |
|---|---|---|---|---|
| 1 | Version control | put all code + config in Git; branches + Pull Request (PR) reviews; protect `main` | Git, GitHub | single source of truth with history and rollback; every later step is triggered by Git |
| 2 | Automated tests | write unit tests for key flows; run them with one command | pytest / Jest | a safety net that catches bugs early; automating deploys without tests only ships bugs faster |
| 3 | Containerization | write a Dockerfile; same image on laptop, test and prod | Docker | removes "works on my machine"; gives one versioned artifact to deploy |
| 4 | CI/CD | on every push: build → test → build image → deploy to staging → prod | GitHub Actions | removes manual, error-prone deploys; fast, repeatable releases |
| 5 | Monitoring | collect metrics + logs, dashboards, alerts on errors/downtime | Prometheus, Grafana | find problems before users do; feedback for the next change |

**How the team measures progress** (DORA = DevOps Research and Assessment metrics): deployment frequency ↑, lead time for changes ↓, change failure rate ↓, MTTR (Mean Time To Recovery) ↓.

**🧠 In simple words:** The roadmap moves the team from hand-copied deployments to an automated flow, one step at a time, where each step needs the one before it: Git first because everything starts from code history, tests next so automation has a safety net, Docker so every environment is identical, then a CI/CD pipeline to automate build-test-deploy, and finally monitoring to see how production behaves. In short: Git → tests → Docker → CI/CD → monitoring, each one making the next safe.
