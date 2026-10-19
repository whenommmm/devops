# All 190 question-bank MCQs, with the answer written in

*Read the question, say your answer, then check the **bold** answer. Source: `repo-questions/devops-heros/DevOps-Exam-Practice-Quentions/`.*

## session1-devops-engineer-roadmap
1. What is the primary objective of DevOps?  →  **A. Faster, reliable delivery through collaboration and automation**
2. What does CI mean?  →  **B. Continuous Integration**
3. Which practice helps catch defects early?  →  **D. Automated tests in CI**
4. What is infrastructure as code?  →  **C. Managing infrastructure using versioned configuration**
5. Which tool is commonly used for source control?  →  **C. Git**
6. What does a CI/CD pipeline automate?  →  **A. Build, test and delivery steps**
7. Why are containers useful?  →  **A. They package applications with dependencies**
8. What is monitoring used for?  →  **C. Tracking system health and behavior**
9. Which mindset is central to DevOps?  →  **A. Shared ownership across development and operations**
10. Which is a common DevOps lifecycle activity?  →  **A. Continuous feedback**

## session2-linux
1. Which command prints the working directory?  →  **D. pwd**
2. Which command lists files including hidden files?  →  **C. ls -a**
3. What does chmod change?  →  **A. File permissions**
4. Which command searches text in files?  →  **D. grep**
5. Which command displays a file's contents?  →  **D. cat**
6. What does the pipe operator do?  →  **D. Passes one command's output to another**
7. Which command shows running processes?  →  **C. ps**
8. Which command changes the current directory?  →  **A. cd**
9. What does sudo typically allow?  →  **A. Run commands with elevated privileges**
10. Which command displays disk space usage by filesystem?  →  **D. df -h**

## session3-shell-scripting
1. Which line commonly selects Bash as script interpreter?  →  **C. #!/bin/bash**
2. How do you reference the first positional argument in Bash?  →  **D. $1**
3. Which operator tests whether a regular file exists in Bash?  →  **C. [ -f file ]**
4. What does $? contain immediately after a command?  →  **A. Exit status of the previous command**
5. Which loop iterates over a sequence of values?  →  **B. for**
6. How do you make a script executable?  →  **C. chmod +x script.sh**
7. Which construct makes a conditional decision?  →  **C. if ... then ... fi**
8. How do you assign a variable in Bash?  →  **B. NAME=value**
9. Which command reads input from the terminal?  →  **D. read**
10. What does && mean between two commands?  →  **B. Run the second only if the first succeeds**

## session4-networking
1. What does DNS primarily do?  →  **A. Resolve names to IP addresses**
2. What does an IP address identify?  →  **C. A network interface on an IP network**
3. Which protocol provides reliable ordered delivery?  →  **C. TCP**
4. Which command can test basic ICMP reachability?  →  **A. ping**
5. What does HTTP status 404 mean?  →  **B. Resource not found**
6. What is the usual default port for HTTPS?  →  **C. 443**
7. Which tool can make HTTP requests from a terminal?  →  **D. curl**
8. What does a subnet mask/prefix indicate?  →  **C. The network portion of an IP address**
9. What is the role of a default gateway?  →  **A. Forward traffic to other networks**
10. Which protocol is commonly used for secure remote shell access?  →  **C. SSH**

## session5-git-github
1. Which command initializes a local Git repository?  →  **A. git init**
2. Which command stages a file for commit?  →  **C. git add**
3. Which command records staged changes?  →  **C. git commit**
4. Which command shows the current working tree state?  →  **A. git status**
5. Which command creates a new branch?  →  **B. git branch feature**
6. Which command integrates another branch into the current branch?  →  **C. git merge**
7. What does git clone do?  →  **D. Copies a repository to a local directory**
8. What is a pull request typically used for?  →  **C. Reviewing and discussing changes before merge**
9. What does .gitignore control?  →  **A. Which untracked paths Git should ignore**
10. Which command uploads local commits to a remote?  →  **C. git push**

## session6-7-docker
1. What is a Docker image?  →  **D. A read-only template for creating containers**
2. Which command builds an image from a Dockerfile?  →  **C. docker build**
3. Which Dockerfile instruction sets the base image?  →  **D. FROM**
4. Which Dockerfile instruction copies files into an image?  →  **C. COPY**
5. Which command lists running containers?  →  **B. docker ps**
6. What does docker run -p 8080:80 map?  →  **C. Host port 8080 to container port 80**
7. What does docker logs show?  →  **A. Container standard output and error**
8. Which command executes a command inside a running container?  →  **A. docker exec**
9. What is the purpose of .dockerignore?  →  **C. Exclude paths from the build context**
10. Which Dockerfile instruction defines a default command?  →  **D. CMD**

## session8-docker-networking-volume
1. Why use Docker volumes?  →  **C. Persist data beyond a container lifecycle**
2. Which command lists Docker volumes?  →  **C. docker volume ls**
3. What is a bind mount?  →  **C. A host path mounted inside a container**
4. Which Docker network driver is the typical default for standalone containers?  →  **D. bridge**
5. What does a user-defined bridge network provide?  →  **B. Container-to-container name resolution**
6. Which flag publishes a container port?  →  **D.  -p **
7. Which command creates a Docker network?  →  **C. docker network create**
8. What is Docker Compose mainly for?  →  **B. Defining and running multi-container applications**
9. Which file is commonly used by Docker Compose?  →  **D. compose.yaml**
10. Which command starts services from a Compose file?  →  **C. docker compose up**

## session9-k8s
1. What is Kubernetes primarily used for?  →  **D. Orchestrating containerized applications**
2. What is a Kubernetes Pod?  →  **C. Smallest deployable unit containing one or more containers**
3. Which component stores Kubernetes cluster state?  →  **C. etcd**
4. Which component schedules Pods onto nodes?  →  **A. kube-scheduler**
5. What does the kubelet do?  →  **D. Ensures Pod containers run on a node**
6. What is kubectl?  →  **B. A CLI for interacting with the Kubernetes API**
7. Which command lists Pods?  →  **D. kubectl get pods**
8. What is a Kubernetes namespace?  →  **B. A logical grouping for cluster resources**
9. Which component exposes the Kubernetes API?  →  **B. kube-apiserver**
10. What is a worker node?  →  **D. A machine that runs application Pods**

## session10-k8s-core-objects
1. What is the purpose of a Deployment?  →  **B. Manage desired replicas and rolling updates**
2. What does a ReplicaSet ensure?  →  **A. A desired number of matching Pods are running**
3. Which field commonly sets Deployment replica count?  →  **D. spec.replicas**
4. What does a Pod label help with?  →  **C. Selecting and organizing resources**
5. What is a label selector used for?  →  **B. Matching resources with specified labels**
6. Which command scales a Deployment?  →  **C. kubectl scale deployment**
7. What happens when a Deployment Pod crashes?  →  **A. A controller works to restore desired state**
8. Which command views rollout status?  →  **B. kubectl rollout status deployment/name**
9. What does a rolling update do?  →  **A. Gradually replace old Pods with new ones**
10. Which resource usually creates and manages ReplicaSets?  →  **D. Deployment**

## session-11-kubernetes-services
1. Why are Kubernetes Services used?  →  **A. Provide stable access to a set of Pods**
2. Which Service type is typically reachable only within the cluster?  →  **A. ClusterIP**
3. Which Service type exposes a port on each node?  →  **A. NodePort**
4. Which Service type commonly provisions an external load balancer on supported clouds?  →  **A. LoadBalancer**
5. How does a Service identify backend Pods?  →  **A. Label selector**
6. What does a Service port targetPort refer to?  →  **C. Port on backend Pods**
7. What is the role of kube-proxy in many clusters?  →  **D. Implement Service traffic routing rules**
8. Which DNS name form can identify a Service across namespaces?  →  **C. service.namespace.svc.cluster.local**
9. What does a headless Service commonly set?  →  **D. clusterIP: None**
10. Which command lists Services?  →  **C. kubectl get svc**

## session-12-ingress-configmaps-secrets
1. What does Ingress commonly define?  →  **A. HTTP/HTTPS routing rules to Services**
2. What is needed to implement Ingress rules?  →  **D. An Ingress controller**
3. What is a ConfigMap intended to store?  →  **D. Non-sensitive configuration data**
4. What is a Secret intended to store?  →  **A. Sensitive configuration values**
5. How can a ConfigMap be consumed by a Pod?  →  **D. Environment variables or mounted files**
6. What does a path rule in Ingress match?  →  **B. The HTTP request path**
7. What does TLS termination usually provide at Ingress?  →  **D. HTTPS encryption for client connections**
8. Which command lists ConfigMaps?  →  **A. kubectl get configmaps**
9. Are Kubernetes Secret values encrypted merely because YAML uses base64?  →  **B. No, base64 is encoding rather than encryption**
10. Which Kubernetes resource commonly holds a TLS certificate and key for Ingress?  →  **C. Secret**

## session-13-storage-hpa-probes
1. What is a PersistentVolumeClaim (PVC)?  →  **A. A request for persistent storage**
2. What is a PersistentVolume (PV)?  →  **B. A cluster storage resource**
3. What does a StorageClass help define?  →  **D. How dynamic storage provisioning occurs**
4. What does HPA stand for?  →  **C. Horizontal Pod Autoscaler**
5. What does an HPA commonly adjust?  →  **B. Number of Pod replicas**
6. What does a readiness probe determine?  →  **B. Whether a container should receive traffic**
7. What does a liveness probe help detect?  →  **D. When a container should be restarted**
8. What does a startup probe help with?  →  **A. Allowing slow-starting apps time to initialize**
9. What can happen if a Pod's PVC remains unbound?  →  **B. The Pod may remain Pending**
10. Which command lists PVCs?  →  **D. kubectl get pvc**

## session-14-kubernetes-troubleshooting
1. Which command gives a quick overview of Pod status?  →  **A. kubectl get pods**
2. Which command displays detailed Pod events and conditions?  →  **D. kubectl describe pod**
3. Which command reads a container's logs?  →  **D. kubectl logs**
4. Which command runs a command inside a running container?  →  **A. kubectl exec**
5. What often causes ImagePullBackOff?  →  **D. An image cannot be downloaded**
6. What often causes CrashLoopBackOff?  →  **B. A container repeatedly exits or crashes**
7. What is a common reason for a Pod to remain Pending?  →  **D. Insufficient schedulable resources**
8. Where can you see recent scheduling or pull errors?  →  **A. Kubernetes Events**
9. What should you check if a Service has no endpoints?  →  **B. Whether its selector matches ready Pods**
10. What is the final step after fixing an incident?  →  **C. Verify the workload behaves correctly**

## session-15-helm
1. What is Helm?  →  **A. A package manager for Kubernetes**
2. What is a Helm chart?  →  **D. A package of Kubernetes templates and metadata**
3. What is Chart.yaml used for?  →  **C. Chart metadata such as name and version**
4. What is values.yaml used for?  →  **A. Default configurable values for templates**
5. Which command creates a starter chart?  →  **B. helm create**
6. Which command installs a chart release?  →  **A. helm install**
7. Which command updates an installed release?  →  **C. helm upgrade**
8. Which command returns to an earlier release revision?  →  **C. helm rollback**
9. Which flag can override a value on the command line?  →  **B. --set**
10. Which command renders templates without installing?  →  **A. helm template**

## session-16-github-actions
1. Where are GitHub Actions workflow files normally stored?  →  **A. .github/workflows/**
2. Which YAML key specifies workflow triggers?  →  **C. on**
3. What is a job in GitHub Actions?  →  **C. A group of steps run on a runner**
4. What does runs-on specify?  →  **A. Runner environment for a job**
5. Which action commonly checks out repository code?  →  **A. actions/checkout**
6. What does a step's run field do?  →  **A. Executes shell commands**
7. What does a step's uses field do?  →  **C. Invokes a reusable action**
8. How should sensitive tokens be supplied to workflows?  →  **A. GitHub Actions secrets**
9. What does needs do between jobs?  →  **D. Declare job dependencies**
10. Which trigger runs a workflow on code pushes?  →  **A. push**

## session-17-devsecops
1. What is DevSecOps?  →  **D. Integrating security throughout development and delivery**
2. What does SAST analyze?  →  **A. Source code or binaries without executing the application**
3. What does DAST test?  →  **C. A running application from the outside**
4. What does dependency scanning identify?  →  **A. Known vulnerabilities in third-party packages**
5. What does container image scanning inspect?  →  **B. Image packages and known vulnerabilities**
6. What is the principle of least privilege?  →  **B. Grant only permissions required**
7. Why should secrets not be committed to Git?  →  **A. They may be exposed through repository history**
8. What is an SBOM?  →  **A. Software Bill of Materials listing components**
9. What is a common purpose of a security gate in CI?  →  **B. Block promotion when policy checks fail**
10. What does shift-left security mean?  →  **A. Run security checks earlier in development**

## session18-terraform-iac
1. What is Terraform used for?  →  **D. Provisioning and managing infrastructure as code**
2. Which command initializes providers and backend?  →  **D. terraform init**
3. Which command previews infrastructure changes?  →  **B. terraform plan**
4. Which command applies the planned changes?  →  **C. terraform apply**
5. What does Terraform state track?  →  **C. Mappings between configuration and managed resources**
6. What is a Terraform provider?  →  **B. Plugin for interacting with an API or platform**
7. What is a Terraform module?  →  **A. Reusable group of Terraform configuration**
8. Which command checks Terraform configuration syntax and consistency?  →  **D. terraform validate**
9. What is a variable in Terraform?  →  **B. An input used to parameterize configuration**
10. Why use a remote backend with locking?  →  **A. Share state safely and reduce concurrent update conflicts**

## session19-cloud-terraform
1. What does IaaS provide?  →  **A. Virtualized infrastructure such as compute and networks**
2. What does PaaS primarily provide?  →  **C. Managed platform for deploying applications**
3. What does SaaS typically deliver?  →  **C. Ready-to-use software over a network**
4. What is a cloud region?  →  **A. A geographic area containing cloud infrastructure**
5. What is an availability zone?  →  **A. An isolated infrastructure location within a region**
6. What is a VPC?  →  **A. A logically isolated virtual network**
7. What is a subnet?  →  **C. An IP address range within a network**
8. What does a route table define?  →  **A. Where network traffic should be forwarded**
9. What does a security group commonly control?  →  **D. Allowed inbound and outbound network traffic**
10. Why combine Terraform with cloud services?  →  **A. Reproducible, versioned infrastructure provisioning**

## session20-monitoring-observability-gitops
1. What is a metric?  →  **D. A numeric measurement over time**
2. What is a log?  →  **D. A timestamped record of events**
3. What is a distributed trace?  →  **A. A view of a request across services**
4. What is Prometheus commonly used for?  →  **C. Collecting and querying time-series metrics**
5. What is Grafana commonly used for?  →  **A. Visualizing metrics and creating dashboards**
6. What does observability help engineers do?  →  **B. Infer internal system behavior from telemetry**
7. What is GitOps?  →  **D. Managing desired system state through version-controlled Git**
8. In GitOps, what is the source of truth?  →  **D. The desired configuration stored in Git**
9. What is Argo CD commonly used for?  →  **A. Reconciling Kubernetes deployments with Git**
10. What does configuration drift mean?  →  **A. Actual state differs from declared desired state**
