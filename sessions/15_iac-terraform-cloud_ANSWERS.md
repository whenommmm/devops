# Answer key · Session 15 · Infrastructure as Code, Terraform & Cloud

⚠️ Only open after answering.

---

## Warm-up

**W1.** Argo CD manages the Deployment `web` with **auto-sync and self-heal turned on**. Git says `replicas: 2`. An engineer runs `kubectl scale deployment web --replicas=5`. What happens a few minutes later?
- A) It stays at 5, as kubectl changes win over Git
- B) Argo CD writes replicas: 5 back into the Git repo
- C) Argo CD reverts it to 2 to match the Git repo
- D) Argo CD deletes and recreates the whole app

**Correct: C.** In GitOps, **Git is the source of truth**. With self-heal on, Argo CD notices the live state (5) has drifted from Git (2) and syncs it back to 2. A is the opposite of GitOps. B: Argo CD **pulls from** Git; it never commits cluster changes back into it. D: it only changes what differs (the replica count), it doesn't recreate the app. *(Without self-heal, Argo CD would only mark the app **OutOfSync** and wait.)*

---

**W2.** Prometheus's **Targets** page shows the counter app as **UP**, and the query on Prometheus's own page returns numbers. But the Grafana panel for the same query says **"No data"**. Where is the problem most likely?
- A) The app stopped exposing its /metrics endpoint
- B) Grafana's data source or panel query is wrong
- C) Prometheus can't reach the app to scrape it
- D) The app must push metrics to Grafana directly

**Correct: B.** The data clearly reaches Prometheus (target UP, query returns values), so the break is **between Prometheus and Grafana**: a wrong data-source URL, a typo in the panel's query, or the wrong time range. A and C would make the target **DOWN** and the Prometheus query empty. D: Grafana stores no metrics itself; it **queries** Prometheus, and Prometheus **pulls** (scrapes) from apps.

---

**W3.** What does this print (three lines)?
```bash
ENV=prod
echo 'Deploying to $ENV'
echo "Deploying to $ENV"
echo Deploying   to   $ENV
```
- A) `Deploying to prod` / `Deploying to prod` / `Deploying   to   prod`
- B) `Deploying to $ENV` / `Deploying to prod` / `Deploying   to   prod`
- C) `Deploying to $ENV` / `Deploying to $ENV` / `Deploying to prod`
- D) `Deploying to $ENV` / `Deploying to prod` / `Deploying to prod`

**Correct: D.** Line 1: **single** outer quotes → everything literal, so `$ENV` is printed as typed. Line 2: **double** quotes → `$ENV` expands to `prod`. Line 3: **no quotes** → `$ENV` expands, and the shell **splits the line into separate words at the spaces**; `echo` then prints its words with **one** space between them, so the extra spaces vanish. B is the near-miss: it forgets that unquoted text loses its extra spaces. A forgets single quotes are literal. C treats double quotes as literal.

---

## Check 86

**86.1** Which command can be run three times in a row and leave **exactly the same result** as running it once?
- A) `echo "export X=1" >> ~/.bashrc`
- B) `git commit --allow-empty -m run`
- C) `kubectl apply -f deploy.yaml`
- D) `docker run -d --rm nginx:1.27`

**Correct: C.** `kubectl apply` is **declarative**: the 2nd and 3rd runs find the Deployment already matching the file and report `unchanged`. That's idempotency. A **appends** a duplicate line every time (`>>`). B creates a **new commit** on every run. D starts a **new container** on every run (3 runs = 3 containers).

---

**86.2** A team must **(1)** create a VPC, three virtual machines, and a load balancer on AWS, then **(2)** install nginx and copy config files onto those machines. Which split matches the usual tool roles?
- A) Terraform for (1), Ansible for (2)
- B) Ansible for (1), then Terraform for (2)
- C) CloudFormation alone for (1) and (2)
- D) Terraform alone for (1) and (2)

**Correct: A.** (1) is **provisioning** (creating infrastructure): Terraform's job. (2) is **configuration management** (setting up software inside existing servers): Ansible's job. B swaps the roles. C: CloudFormation is a provisioning tool; it isn't built for configuring software inside servers. D: Terraform *can* run scripts at boot, but configuring servers isn't its strength; the usual answer splits the jobs.

---

**86.3** Declarative vs imperative
**Model answer:** *"Declarative code describes the desired end result and lets the tool work out the steps (e.g. a Terraform file saying 'two servers exist', or `kubectl apply -f`), whereas imperative code lists the exact steps to run in order (e.g. a shell script that creates one server and then another, or `kubectl scale`)."*

---

## Check 87

**87.1** A VPC is declared as `resource "aws_vpc" "main" { … }`. Inside `resource "aws_subnet" "public"`, what should `vpc_id =` be set to?
- A) `resource.aws_vpc.main`
- B) `var.aws_vpc.main.id`
- C) `aws_vpc["main"].id`
- D) `aws_vpc.main.id`

**Correct: D.** A resource is referenced as **TYPE.LOCAL_NAME.ATTRIBUTE**, with no prefix. A adds a `resource.` prefix that doesn't exist and has no attribute. B: `var.` is for **input variables** only. C uses the `[ ]` indexing that's only for resources made with `count` or `for_each`.

---

**87.2** How does Terraform decide the order in which to create resources?
- A) It follows the order the blocks appear in the files
- B) It builds a graph from references between resources
- C) It creates resources alphabetically by their type
- D) It needs depends_on on every resource that has a parent

**Correct: B.** References like `vpc_id = aws_vpc.main.id` create **implicit dependencies**, and Terraform builds a **dependency graph** from them (independent resources are created in parallel; destroy runs in reverse). A: file order and block order don't matter; all `.tf` files are read together. C: alphabetical order plays no part. D: `depends_on` is only for the rare **hidden** dependency that no reference shows.

---

**87.3** `resource` vs `data`
**Model answer:** *"A resource block creates and manages something (Terraform will create, update, and destroy it, and records it in the state), whereas a data block only reads information about something that already exists (such as the newest Ubuntu AMI) and creates nothing."*

---

## Check 88

**88.1** `terraform plan` shows `-/+ resource "aws_instance" "web"` with `ami = "ami-0aaa" -> "ami-0bbb" # forces replacement`. What will `apply` do to the server?
- A) Change the AMI on the running instance in place
- B) Create a second instance and keep both running
- C) Delete the instance, then create a new one
- D) Nothing, since -/+ only marks a dry-run preview

**Correct: C.** `-/+` = **destroy, then create a replacement**. An instance's machine image can't be swapped on a running VM, so it's rebuilt (new ID, new IP, and anything stored on the old disk is gone). A describes `~` (update in place). B: the old one is deleted; nothing is duplicated (with `create_before_destroy` the new one is merely created *first*, shown as `+/-`). D: `plan` is the dry run, but the symbols describe what `apply` **will** do.

---

**88.2** A teammate wants to commit `terraform.tfstate` to the team's GitHub repo "so everyone has the latest copy". What's the best response?
- A) No: use a remote backend with locking instead
- B) Fine, as long as the repo is private to the team
- C) Fine, after running `terraform fmt` on the state
- D) No: delete the state and rely on `terraform plan`

**Correct: A.** The state holds **secrets in plain text**, and copies in Git go stale and conflict, with **no locking** to stop two simultaneous applies. A **remote backend** (e.g. S3 with locking, or HCP Terraform) gives everyone one shared, locked, encrypted state. B: "private" still puts secrets in Git history forever, and doesn't solve locking. C: `fmt` formats `.tf` files; it has nothing to do with the state. D: without state, Terraform forgets what it owns and would try to create everything again.

---

**88.3** You apply a security group with one inbound rule. A colleague then adds a second inbound rule **by hand in the AWS console**. What does your next `terraform plan` show?
- A) No changes, since state still matches your code
- B) An update that adds the new rule into your .tf file
- C) An error, because the state file is now corrupted
- D) An update that removes the rule added by hand

**Correct: D.** `plan` **refreshes** first: it reads the real security group, sees an extra rule the code doesn't have (**drift**), and plans an update (`~`) to remove it, because the code is the source of truth. A: plan compares with the **real** infrastructure, not just the old state. B: Terraform **never edits your `.tf` files**. C: a manual change is normal drift, not corruption. *(To keep the rule, add it to the code instead.)*

---

## Check 89

**89.1** A startup pushes its code to a platform that builds and runs it. They never patch an operating system or manage servers, but they do write and maintain the app. Which model is this?
- A) IaaS, like renting EC2 virtual machines
- B) SaaS, like Gmail or Salesforce for users
- C) PaaS, like Heroku or Elastic Beanstalk
- D) On-premises, like servers in your office

**Correct: C.** **PaaS**: you bring the code; the provider runs the OS, runtime, and servers ("pizza delivery"). A: with IaaS **you** patch the OS on your VMs. B: with SaaS you don't write the app at all; you just use finished software. D: on-premises means you own and manage everything, including the hardware.

---

**89.2** A security group allows **inbound TCP 80 from 0.0.0.0/0** and has **no outbound rules at all**. A user's browser requests a page from the instance. What happens to the response?
- A) It's blocked, as there's no outbound rule for it
- B) It's allowed, as security groups are stateful
- C) It's allowed only if the reply also uses port 80
- D) It's blocked unless the instance uses a NAT gateway

**Correct: B.** Security groups are **stateful**: they remember the allowed incoming connection, so its **reply** goes out automatically, with no outbound rule needed. A is how a **stateless NACL** behaves. C: the reply goes to the browser's **ephemeral port** (1024–65535), not port 80, and a stateful SG doesn't care either way. D: a NAT gateway is for connections that private servers **start**, not for replies. *(Without outbound rules, the server couldn't **start** its own connections, e.g. to download updates.)*

---

**89.3** Region vs AZ
**Model answer:** *"A region is a separate geographic area (like ap-south-1, Mumbai) containing several Availability Zones, whereas an Availability Zone is one or more data centres inside that region with their own power, cooling, and network. Spreading servers across two AZs gives high availability: if one AZ fails, the servers in the other keep serving users."*

---

## Session quiz

**Q1.** After a clean apply, you change **two things**: (1) you add a tag to `aws_vpc.main`, and (2) you change the `ami` of `aws_instance.web` (which forces replacement). Which summary line does `terraform plan` print?
- A) `Plan: 0 to add, 2 to change, 0 to destroy.`
- B) `Plan: 1 to add, 1 to change, 1 to destroy.`
- C) `Plan: 2 to add, 0 to change, 2 to destroy.`
- D) `Plan: 1 to add, 2 to change, 1 to destroy.`

**Correct: B.** The tag is an **in-place update** (`~`): **1 to change**. The AMI change is a **replacement** (`-/+`), which counts as **1 to add + 1 to destroy**, not as a change. A treats the replacement as an update. C treats the tag change as a replacement too. D counts the replacement **twice** (as a change *and* as add/destroy).

---

**Q2.** Two engineers work on the same Terraform code from Git, and **each keeps `terraform.tfstate` on their own laptop**. Engineer A runs `apply` and creates a VPC. Engineer B, with an up-to-date checkout, has never applied. B runs `terraform plan`. What does B see?
- A) No changes, because the code in Git is identical
- B) A lock error, because A's state is still locked
- C) An error saying the VPC is owned by engineer A
- D) A plan to create the VPC, as B's state is empty

**Correct: D.** Terraform knows what it manages **only through its state**. B's state is empty, so to B's Terraform the VPC "doesn't exist yet", and it plans `+ create`. Applying it would build a **duplicate** VPC (or fail for things with unique names, like S3 buckets). This is exactly why teams use a **shared remote backend**. A: the code being identical doesn't matter; the states differ. B: local state locks are per-laptop files, and A's run has finished anyway. C: Terraform has no idea who "owns" a cloud resource.

---

**Q3.** `variable "env"` has `default = "dev"`. The folder contains `terraform.tfvars` with `env = "staging"`. You run `export TF_VAR_env=qa`, then `terraform apply` with **no** `-var` flags. Which value does `var.env` get?
- A) `dev`, as the default in the variable block wins
- B) `qa`, as environment variables override files
- C) `staging`, as terraform.tfvars beats TF_VAR_
- D) It asks, because three values are in conflict

**Correct: C.** Precedence, weakest to strongest: **default → `TF_VAR_` env var → `terraform.tfvars` → `*.auto.tfvars` → `-var`/`-var-file`**. The strongest value given here is `terraform.tfvars`. (This was checked on Terraform 1.16: the result is `staging`. Add `-var env=prod` and it becomes `prod`.) A: the default is the **weakest**. B is the tempting misconception: env vars beat only the default. D: Terraform never asks when a value exists; it simply uses the strongest one.

---

**Q4.** An EC2 instance in subnet `10.0.1.0/24` has a **public IP**. Its security group allows TCP 80 from `0.0.0.0/0`, and nginx is running and listening on `0.0.0.0:80`. The subnet's route table contains **only** `10.0.0.0/16 → local`. A browser request to the public IP **times out**. What's needed?
- A) Route 0.0.0.0/0 to an internet gateway
- B) An inbound NACL rule that allows port 80
- C) An outbound security group rule for port 80
- D) Move the instance to a subnet in another AZ

**Correct: A.** With only the `local` route, the subnet is effectively **private**: traffic to and from the internet has no path. Adding `0.0.0.0/0 → igw-…` (with an internet gateway attached to the VPC) makes it a **public subnet**. B: the **default NACL already allows everything**, and nothing suggests a custom NACL. C: security groups are **stateful**, so replies need no outbound rule. D: the AZ has nothing to do with internet routing.

---

**L1.** Full forms + what `init` creates
**Model answer:** *"IaC = Infrastructure as Code; HCL = HashiCorp Configuration Language; VPC = Virtual Private Cloud; IAM = Identity and Access Management. `terraform init` downloads the providers into the hidden `.terraform/` folder and creates the dependency lock file `.terraform.lock.hcl` (it also sets up the backend)."*

**L2.** Security group vs NACL
**Model answer:** *"A security group is a stateful firewall attached to individual instances that has allow rules only, so replies to allowed traffic are let through automatically, whereas a NACL is a stateless firewall for a whole subnet that has numbered allow and deny rules checked in order, so return traffic (ephemeral ports) must be allowed explicitly."*
