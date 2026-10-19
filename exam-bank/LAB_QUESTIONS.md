# Lab QUESTIONS · all 57 labs, in session order

*Write your commands/config in the empty box under each lab, then check [LAB_ANSWERS.md](LAB_ANSWERS.md) (same session and lab number).*


## Session 1 · DevOps intro & roadmap

### Lab 1
> Given a sample application scenario, draw a simple CI/CD workflow from code commit to deployment. Label the build, test, package, deploy, and monitoring stages.

```bash



```

### Lab 2
> Compare a monolithic application and a microservices application for a provided online shopping use case. Identify two operational benefits and two challenges of microservices.

```bash



```

### Lab 3
> Create a one-page DevOps roadmap for a small team that currently deploys manually. Prioritize version control, automated tests, containerization, CI/CD, and monitoring, with a short reason for each.

```bash



```


## Session 2 · Linux

### Lab 1
> Create a project directory with app, logs, and config subdirectories. Create sample files and demonstrate listing, copying, moving, searching, and deleting files.

```bash



```

### Lab 2
> Create a user named examuser and a group named devops. Add the user to the group and configure suitable ownership and permissions on a project directory.

```bash



```

### Lab 3
> Check the status of a Linux service, inspect its logs, and document the commands used to diagnose a simulated service problem.

```bash



```


## Session 3 · Shell scripting

### Lab 1
> Write a shell script that accepts a directory path as an argument and displays the number of files and subdirectories. Handle a missing or invalid argument.

```bash



```

### Lab 2
> Write a script that checks whether a supplied file exists and is readable, writable, and executable. Print a clear result for each check.

```bash



```

### Lab 3
> Write a reusable system-health script using variables, conditions, loops, and functions. Report disk usage, memory, and uptime.

```bash



```


## Session 4 · Networking

### Lab 1
> Use Linux commands to display the machine IP address, subnet information, default route, and listening ports. Include the command and a brief interpretation for each.

```bash



```

### Lab 2
> Use ping, ss, and curl to troubleshoot a web service. Determine whether a simulated failure is caused by reachability, a closed port, or an HTTP response.

```bash



```

### Lab 3
> Resolve a domain using nslookup or dig, then use curl to verify HTTP/HTTPS connectivity. Record the DNS result and HTTP status.

```bash



```


## Session 5 · Git & GitHub

### Lab 1
> Create a Git repository for a small application, add a README, make at least three meaningful commits, and push it to GitHub.

```bash



```

### Lab 2
> Create a feature branch, implement a change, push the branch, and merge it into main through a Pull Request after reviewing the diff.

```bash



```

### Lab 3
> Create a merge conflict by editing the same line on two branches. Resolve it, commit the resolution, and verify the final history.

```bash



```


## Session 6–7 · Docker & Dockerfile

### Lab 1
> Run an Nginx container with host port 8080 mapped to container port 80. Verify access using curl and inspect the container logs.

```bash



```

### Lab 2
> Create a Dockerfile for a simple HTML page using FROM, WORKDIR, COPY, and CMD. Build the image, run it, and verify the page.

```bash



```

### Lab 3
> Add a .dockerignore file and an environment variable to the Docker build. Rebuild and run the image, then explain the purpose of each added instruction.

```bash



```


## Session 8 · Docker networking, volumes & Compose

### Lab 1
> Create a custom Docker network and run two containers on it. Demonstrate name-based communication between the containers.

```bash



```

### Lab 2
> Run a database container with a named volume, create sample data, remove and recreate the container, and verify that the data persists.

```bash



```

### Lab 3
> Write a compose.yaml file for a web application and database. Start the stack, verify service communication, inspect logs, and shut it down.

```bash



```


## Session 9 · Kubernetes fundamentals

### Lab 1
> Connect to a Kubernetes cluster and inspect nodes, namespaces, and cluster resources using kubectl. Record what each command shows.

```bash



```

### Lab 2
> Deploy a simple web application and inspect it using kubectl get, describe, and logs. Confirm that the workload reaches a healthy state.

```bash



```

### Lab 3
> Identify the role of the API Server, Scheduler, Controller Manager, etcd, kubelet, and kubectl in a short cluster architecture diagram.

```bash



```


## Session 10 · Pods, ReplicaSets & Deployments

### Lab 1
> Create a Pod running Nginx and inspect its status, labels, IP address, and logs using kubectl.

```bash



```

### Lab 2
> Create a Deployment with two replicas, scale it to five, and then scale it back to two. Verify the replica count at each step.

```bash



```

### Lab 3
> Deploy version 1 of an application, perform a rolling update to version 2, verify the rollout, and roll back to version 1.

```bash



```


## Session 11 · Kubernetes Services

### Lab 1
> Deploy an application with multiple replicas and create a ClusterIP Service that selects the correct Pods. Verify its endpoints.

```bash



```

### Lab 2
> Create a NodePort Service and demonstrate access to the application through the exposed node port.

```bash



```

### Lab 3
> Configure a Service whose port and targetPort differ. Verify that traffic reaches the intended container port.

```bash



```


## Session 12 · Ingress, ConfigMaps & Secrets

### Lab 1
> Deploy an application and configure an Ingress rule that routes requests by host or URL path to the correct Service.

```bash



```

### Lab 2
> Create a ConfigMap and inject its values into a Deployment as environment variables. Verify the values inside a running Pod.

```bash



```

### Lab 3
> Create a Kubernetes Secret for a sample credential and consume it from a Pod without hard-coding the credential in the Deployment manifest.

```bash



```


## Session 13 · Storage, HPA & Probes

### Lab 1
> Create a PersistentVolumeClaim, mount it into a Pod, write a file, recreate the Pod, and verify that the file remains.

```bash



```

### Lab 2
> Configure readiness and liveness probes for a sample application. Demonstrate how Kubernetes reacts when the application is not ready or becomes unhealthy.

```bash



```

### Lab 3
> Configure an HPA for a Deployment with CPU requests defined. Generate load and observe replica scaling; document any cluster prerequisites.

```bash



```


## Session 14 · Kubernetes troubleshooting

### Lab 1
> Deploy a workload with an invalid image. Diagnose ImagePullBackOff using kubectl get, describe, and events, then correct the image.

```bash



```

### Lab 2
> Diagnose a CrashLoopBackOff workload using logs and describe. Identify the root cause, fix the configuration, and verify recovery.

```bash



```

### Lab 3
> Create a Service with an incorrect selector. Use labels, selectors, endpoints, and Service configuration to identify and fix the issue.

```bash



```


## Session 15 · Helm

### Lab 1
> Create a Helm chart for a simple web application. Explain the purpose of Chart.yaml, values.yaml, and the templates directory.

```bash



```

### Lab 2
> Deploy the chart with configurable image, replica count, and Service port values. Verify the rendered and installed Kubernetes resources.

```bash



```

### Lab 3
> Upgrade a Helm release, inspect its revision history, and roll back to the previous revision. Verify the final release status.

```bash



```


## Session 16 · CI/CD & GitHub Actions

### Lab 1
> Create a GitHub Actions workflow triggered on push that checks out the repository, installs dependencies, and builds the application.

```bash



```

### Lab 2
> Add an automated test step. Demonstrate that a failing test causes the workflow to fail and that a corrected test passes.

```bash



```

### Lab 3
> Extend the workflow to upload a build artifact and make it downloadable from the Actions run.

```bash



```


## Session 17 · Complete CI/CD & DevSecOps

### Lab 1
> Create a CI/CD pipeline that builds a Docker image and pushes it to a container registry using repository secrets rather than hard-coded credentials.

```bash



```

### Lab 2
> Add SAST, dependency scanning, and secret scanning to a pipeline. Configure a required security check to block later stages when it fails.

```bash



```

### Lab 3
> Add container image vulnerability scanning and implement a gate that prevents deployment when the agreed severity threshold is exceeded.

```bash



```


## Session 18 · Terraform & IaC

### Lab 1
> Write Terraform configuration that declares a provider and creates a small supported resource. Run fmt, init, validate, and plan, then explain the output.

```bash



```

### Lab 2
> Use variables and outputs in a Terraform configuration. Pass a variable through a .tfvars file and print a useful output after apply.

```bash



```

### Lab 3
> Modify a resource configuration, inspect the plan before applying, and explain how Terraform state maps configuration to real infrastructure. Do not commit secrets or state files.

```bash



```


## Session 19 · Cloud & Terraform in action

### Lab 1
> Design a cloud network with a VPC/VNet, public and private subnets, route tables, and security rules. Submit a diagram and explain the traffic flow.

```bash



```

### Lab 2
> Use Terraform to provision a small cloud network and one low-cost compute resource in a sandbox account. Run plan before apply and destroy the lab resources afterward.

```bash



```

### Lab 3
> Configure a security group/firewall rule that permits only the required application port. Demonstrate that SSH/admin access is not open to the entire internet.

```bash



```


## Session 20 · Monitoring, Observability & GitOps

### Lab 1
> Expose application metrics and configure Prometheus to scrape them. Verify the target is healthy and identify at least two metrics.

```bash



```

### Lab 2
> Create a Grafana dashboard with panels for request rate, errors, and latency, or use the metrics available in the supplied environment. Explain what each panel indicates.

```bash



```

### Lab 3
> Configure a basic GitOps deployment using Argo CD or a supplied GitOps environment. Change the desired replica count in Git and verify reconciliation in the cluster.

```bash



```

