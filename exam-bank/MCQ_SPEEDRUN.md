# ⚡ MCQ speedrun · 190 questions, one line each

*Read fast: question → **answer**. ~15 minutes for all of them. Then quiz yourself in [MCQ_BANK.md](MCQ_BANK.md).*

### 1 · DevOps intro & roadmap
1. What is the primary objective of DevOps? → **Faster, reliable delivery through collaboration and automation**  
2. What does CI mean? → **Continuous Integration**  
3. Which practice helps catch defects early? → **Automated tests in CI**  
4. What is infrastructure as code? → **Managing infrastructure using versioned configuration**  
5. Which tool is commonly used for source control? → **Git**  
6. What does a CI/CD pipeline automate? → **Build, test and delivery steps**  
7. Why are containers useful? → **They package applications with dependencies**  
8. What is monitoring used for? → **Tracking system health and behavior**  
9. Which mindset is central to DevOps? → **Shared ownership across development and operations**  
10. Which is a common DevOps lifecycle activity? → **Continuous feedback**  

### 2 · Linux
1. Which command prints the working directory? → **pwd**  
2. Which command lists files including hidden files? → **ls -a**  
3. What does chmod change? → **File permissions**  
4. Which command searches text in files? → **grep**  
5. Which command displays a file's contents? → **cat**  
6. What does the pipe operator do? → **Passes one command's output to another**  
7. Which command shows running processes? → **ps**  
8. Which command changes the current directory? → **cd**  
9. What does sudo typically allow? → **Run commands with elevated privileges**  
10. Which command displays disk space usage by filesystem? → **df -h**  

### 3 · Shell scripting
1. Which line commonly selects Bash as script interpreter? → **#!/bin/bash**  
2. How do you reference the first positional argument in Bash? → **$1**  
3. Which operator tests whether a regular file exists in Bash? → **[ -f file ]**  
4. What does $? contain immediately after a command? → **Exit status of the previous command**  
5. Which loop iterates over a sequence of values? → **for**  
6. How do you make a script executable? → **chmod +x script.sh**  
7. Which construct makes a conditional decision? → **if ... then ... fi**  
8. How do you assign a variable in Bash? → **NAME=value**  
9. Which command reads input from the terminal? → **read**  
10. What does && mean between two commands? → **Run the second only if the first succeeds**  

### 4 · Networking
1. What does DNS primarily do? → **Resolve names to IP addresses**  
2. What does an IP address identify? → **A network interface on an IP network**  
3. Which protocol provides reliable ordered delivery? → **TCP**  
4. Which command can test basic ICMP reachability? → **ping**  
5. What does HTTP status 404 mean? → **Resource not found**  
6. What is the usual default port for HTTPS? → **443**  
7. Which tool can make HTTP requests from a terminal? → **curl**  
8. What does a subnet mask/prefix indicate? → **The network portion of an IP address**  
9. What is the role of a default gateway? → **Forward traffic to other networks**  
10. Which protocol is commonly used for secure remote shell access? → **SSH**  

### 5 · Git & GitHub
1. Which command initializes a local Git repository? → **git init**  
2. Which command stages a file for commit? → **git add**  
3. Which command records staged changes? → **git commit**  
4. Which command shows the current working tree state? → **git status**  
5. Which command creates a new branch? → **git branch feature**  
6. Which command integrates another branch into the current branch? → **git merge**  
7. What does git clone do? → **Copies a repository to a local directory**  
8. What is a pull request typically used for? → **Reviewing and discussing changes before merge**  
9. What does .gitignore control? → **Which untracked paths Git should ignore**  
10. Which command uploads local commits to a remote? → **git push**  

### 6–7 · Docker & Dockerfile
1. What is a Docker image? → **A read-only template for creating containers**  
2. Which command builds an image from a Dockerfile? → **docker build**  
3. Which Dockerfile instruction sets the base image? → **FROM**  
4. Which Dockerfile instruction copies files into an image? → **COPY**  
5. Which command lists running containers? → **docker ps**  
6. What does docker run -p 8080:80 map? → **Host port 8080 to container port 80**  
7. What does docker logs show? → **Container standard output and error**  
8. Which command executes a command inside a running container? → **docker exec**  
9. What is the purpose of .dockerignore? → **Exclude paths from the build context**  
10. Which Dockerfile instruction defines a default command? → **CMD**  

### 8 · Docker networking, volumes & Compose
1. Why use Docker volumes? → **Persist data beyond a container lifecycle**  
2. Which command lists Docker volumes? → **docker volume ls**  
3. What is a bind mount? → **A host path mounted inside a container**  
4. Which Docker network driver is the typical default for standalone containers? → **bridge**  
5. What does a user-defined bridge network provide? → **Container-to-container name resolution**  
6. Which flag publishes a container port? → **-p**  
7. Which command creates a Docker network? → **docker network create**  
8. What is Docker Compose mainly for? → **Defining and running multi-container applications**  
9. Which file is commonly used by Docker Compose? → **compose.yaml**  
10. Which command starts services from a Compose file? → **docker compose up**  

### 9 · Kubernetes fundamentals
1. What is Kubernetes primarily used for? → **Orchestrating containerized applications**  
2. What is a Kubernetes Pod? → **Smallest deployable unit containing one or more containers**  
3. Which component stores Kubernetes cluster state? → **etcd**  
4. Which component schedules Pods onto nodes? → **kube-scheduler**  
5. What does the kubelet do? → **Ensures Pod containers run on a node**  
6. What is kubectl? → **A CLI for interacting with the Kubernetes API**  
7. Which command lists Pods? → **kubectl get pods**  
8. What is a Kubernetes namespace? → **A logical grouping for cluster resources**  
9. Which component exposes the Kubernetes API? → **kube-apiserver**  
10. What is a worker node? → **A machine that runs application Pods**  

### 10 · Pods, ReplicaSets & Deployments
1. What is the purpose of a Deployment? → **Manage desired replicas and rolling updates**  
2. What does a ReplicaSet ensure? → **A desired number of matching Pods are running**  
3. Which field commonly sets Deployment replica count? → **spec.replicas**  
4. What does a Pod label help with? → **Selecting and organizing resources**  
5. What is a label selector used for? → **Matching resources with specified labels**  
6. Which command scales a Deployment? → **kubectl scale deployment**  
7. What happens when a Deployment Pod crashes? → **A controller works to restore desired state**  
8. Which command views rollout status? → **kubectl rollout status deployment/name**  
9. What does a rolling update do? → **Gradually replace old Pods with new ones**  
10. Which resource usually creates and manages ReplicaSets? → **Deployment**  

### 11 · Kubernetes Services
1. Why are Kubernetes Services used? → **Provide stable access to a set of Pods**  
2. Which Service type is typically reachable only within the cluster? → **ClusterIP**  
3. Which Service type exposes a port on each node? → **NodePort**  
4. Which Service type commonly provisions an external load balancer on supported clouds? → **LoadBalancer**  
5. How does a Service identify backend Pods? → **Label selector**  
6. What does a Service port targetPort refer to? → **Port on backend Pods**  
7. What is the role of kube-proxy in many clusters? → **Implement Service traffic routing rules**  
8. Which DNS name form can identify a Service across namespaces? → **service.namespace.svc.cluster.local**  
9. What does a headless Service commonly set? → **clusterIP: None**  
10. Which command lists Services? → **kubectl get svc**  

### 12 · Ingress, ConfigMaps & Secrets
1. What does Ingress commonly define? → **HTTP/HTTPS routing rules to Services**  
2. What is needed to implement Ingress rules? → **An Ingress controller**  
3. What is a ConfigMap intended to store? → **Non-sensitive configuration data**  
4. What is a Secret intended to store? → **Sensitive configuration values**  
5. How can a ConfigMap be consumed by a Pod? → **Environment variables or mounted files**  
6. What does a path rule in Ingress match? → **The HTTP request path**  
7. What does TLS termination usually provide at Ingress? → **HTTPS encryption for client connections**  
8. Which command lists ConfigMaps? → **kubectl get configmaps**  
9. Are Kubernetes Secret values encrypted merely because YAML uses base64? → **No, base64 is encoding rather than encryption**  
10. Which Kubernetes resource commonly holds a TLS certificate and key for Ingress? → **Secret**  

### 13 · Storage, HPA & Probes
1. What is a PersistentVolumeClaim (PVC)? → **A request for persistent storage**  
2. What is a PersistentVolume (PV)? → **A cluster storage resource**  
3. What does a StorageClass help define? → **How dynamic storage provisioning occurs**  
4. What does HPA stand for? → **Horizontal Pod Autoscaler**  
5. What does an HPA commonly adjust? → **Number of Pod replicas**  
6. What does a readiness probe determine? → **Whether a container should receive traffic**  
7. What does a liveness probe help detect? → **When a container should be restarted**  
8. What does a startup probe help with? → **Allowing slow-starting apps time to initialize**  
9. What can happen if a Pod's PVC remains unbound? → **The Pod may remain Pending**  
10. Which command lists PVCs? → **kubectl get pvc**  

### 14 · Kubernetes troubleshooting
1. Which command gives a quick overview of Pod status? → **kubectl get pods**  
2. Which command displays detailed Pod events and conditions? → **kubectl describe pod**  
3. Which command reads a container's logs? → **kubectl logs**  
4. Which command runs a command inside a running container? → **kubectl exec**  
5. What often causes ImagePullBackOff? → **An image cannot be downloaded**  
6. What often causes CrashLoopBackOff? → **A container repeatedly exits or crashes**  
7. What is a common reason for a Pod to remain Pending? → **Insufficient schedulable resources**  
8. Where can you see recent scheduling or pull errors? → **Kubernetes Events**  
9. What should you check if a Service has no endpoints? → **Whether its selector matches ready Pods**  
10. What is the final step after fixing an incident? → **Verify the workload behaves correctly**  

### 15 · Helm
1. What is Helm? → **A package manager for Kubernetes**  
2. What is a Helm chart? → **A package of Kubernetes templates and metadata**  
3. What is Chart.yaml used for? → **Chart metadata such as name and version**  
4. What is values.yaml used for? → **Default configurable values for templates**  
5. Which command creates a starter chart? → **helm create**  
6. Which command installs a chart release? → **helm install**  
7. Which command updates an installed release? → **helm upgrade**  
8. Which command returns to an earlier release revision? → **helm rollback**  
9. Which flag can override a value on the command line? → **--set**  
10. Which command renders templates without installing? → **helm template**  

### 16 · CI/CD & GitHub Actions
1. Where are GitHub Actions workflow files normally stored? → **.github/workflows/**  
2. Which YAML key specifies workflow triggers? → **on**  
3. What is a job in GitHub Actions? → **A group of steps run on a runner**  
4. What does runs-on specify? → **Runner environment for a job**  
5. Which action commonly checks out repository code? → **actions/checkout**  
6. What does a step's run field do? → **Executes shell commands**  
7. What does a step's uses field do? → **Invokes a reusable action**  
8. How should sensitive tokens be supplied to workflows? → **GitHub Actions secrets**  
9. What does needs do between jobs? → **Declare job dependencies**  
10. Which trigger runs a workflow on code pushes? → **push**  

### 17 · Complete CI/CD & DevSecOps
1. What is DevSecOps? → **Integrating security throughout development and delivery**  
2. What does SAST analyze? → **Source code or binaries without executing the application**  
3. What does DAST test? → **A running application from the outside**  
4. What does dependency scanning identify? → **Known vulnerabilities in third-party packages**  
5. What does container image scanning inspect? → **Image packages and known vulnerabilities**  
6. What is the principle of least privilege? → **Grant only permissions required**  
7. Why should secrets not be committed to Git? → **They may be exposed through repository history**  
8. What is an SBOM? → **Software Bill of Materials listing components**  
9. What is a common purpose of a security gate in CI? → **Block promotion when policy checks fail**  
10. What does shift-left security mean? → **Run security checks earlier in development**  

### 18 · Terraform & IaC
1. What is Terraform used for? → **Provisioning and managing infrastructure as code**  
2. Which command initializes providers and backend? → **terraform init**  
3. Which command previews infrastructure changes? → **terraform plan**  
4. Which command applies the planned changes? → **terraform apply**  
5. What does Terraform state track? → **Mappings between configuration and managed resources**  
6. What is a Terraform provider? → **Plugin for interacting with an API or platform**  
7. What is a Terraform module? → **Reusable group of Terraform configuration**  
8. Which command checks Terraform configuration syntax and consistency? → **terraform validate**  
9. What is a variable in Terraform? → **An input used to parameterize configuration**  
10. Why use a remote backend with locking? → **Share state safely and reduce concurrent update conflicts**  

### 19 · Cloud & Terraform in action
1. What does IaaS provide? → **Virtualized infrastructure such as compute and networks**  
2. What does PaaS primarily provide? → **Managed platform for deploying applications**  
3. What does SaaS typically deliver? → **Ready-to-use software over a network**  
4. What is a cloud region? → **A geographic area containing cloud infrastructure**  
5. What is an availability zone? → **An isolated infrastructure location within a region**  
6. What is a VPC? → **A logically isolated virtual network**  
7. What is a subnet? → **An IP address range within a network**  
8. What does a route table define? → **Where network traffic should be forwarded**  
9. What does a security group commonly control? → **Allowed inbound and outbound network traffic**  
10. Why combine Terraform with cloud services? → **Reproducible, versioned infrastructure provisioning**  

### 20 · Monitoring, Observability & GitOps
1. What is a metric? → **A numeric measurement over time**  
2. What is a log? → **A timestamped record of events**  
3. What is a distributed trace? → **A view of a request across services**  
4. What is Prometheus commonly used for? → **Collecting and querying time-series metrics**  
5. What is Grafana commonly used for? → **Visualizing metrics and creating dashboards**  
6. What does observability help engineers do? → **Infer internal system behavior from telemetry**  
7. What is GitOps? → **Managing desired system state through version-controlled Git**  
8. In GitOps, what is the source of truth? → **The desired configuration stored in Git**  
9. What is Argo CD commonly used for? → **Reconciling Kubernetes deployments with Git**  
10. What does configuration drift mean? → **Actual state differs from declared desired state**  

