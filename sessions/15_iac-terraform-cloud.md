# Session 15 · Infrastructure as Code, Terraform & Cloud (Oct 18, Session 1)

**Time:** ~120–140 min (two lectures in one) · **Concepts:** 86 → 89 · **Answers:** `MY_ANSWERS.md` or say "Terraform questions" · **Answer key:** `15_iac-terraform-cloud_ANSWERS.md`

**How to read this:** two lectures in one ("Terraform & IaC" and "Cloud & Terraform in Action"). New words are **bold** and explained where they first appear. No cloud account needed: Practical A runs only on your Mac; Practical B uses **LocalStack** (a pretend Amazon cloud, already running).
- **Before you start:** `terraform version` should print `Terraform v1.16.x on darwin_arm64` (an "out of date!" note is harmless),
- and `docker ps --filter name=localstack --format '{{.Names}}  {{.Status}}'` should print `localstack  Up … (healthy)` (if Colima is stopped, run `colima start` first).
> ⚠️ **Never delete the `minikube` or `localstack` containers.** Terraform in this session only ever deletes things **it created itself**, so it can't touch them.

## 🔁 Warm-up (from Session 14 and the weak list)

**W1.** Argo CD manages the Deployment `web` with **auto-sync and self-heal turned on**. Git says `replicas: 2`. An engineer runs `kubectl scale deployment web --replicas=5`. What happens a few minutes later?
- A) It stays at 5, as kubectl changes win over Git
- B) Argo CD writes replicas: 5 back into the Git repo
- C) Argo CD reverts it to 2 to match the Git repo
- D) Argo CD deletes and recreates the whole app

**W2.** Prometheus's **Targets** page shows the counter app as **UP**, and running the query on Prometheus's own page returns numbers. But the Grafana panel for the same query says **"No data"**. Where is the problem most likely?
- A) The app stopped exposing its /metrics endpoint
- B) Grafana's data source or panel query is wrong
- C) Prometheus can't reach the app to scrape it
- D) The app must push metrics to Grafana directly

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

## Concept 86 · Infrastructure as Code: writing your servers down

**Words you'll learn:** infrastructure, provision, ClickOps, snowflake server, configuration drift, IaC, declarative, imperative, idempotent, mutable / immutable, provisioning vs configuration management, agent, agentless, Terraform, CloudFormation, Ansible, Chef, Puppet.

**The problem.** **Infrastructure** = the servers, networks, firewalls, databases, load balancers, and DNS records an app runs on; to **provision** = to create and set them up.
- Asha provisions the counter app's cloud by clicking in the **cloud console** (the provider's website). That's **ClickOps**: slow, unrepeatable, no record.
- Six months later nobody can build an identical test copy, a port opened by hand at 2 a.m. is still open, and there's no history of who changed what.
- The servers have become **snowflake servers** (hand-patched until unique, so nobody dares rebuild them), and **configuration drift** has set in (the real setup slowly drifts away from what's intended).

**The idea.** **Infrastructure as Code (IaC)** = describing infrastructure in text files and letting a tool create, change, and delete the real thing to match (a Dockerfile, but for what's underneath).
- The files live in **Git**: history, review in PRs, identical dev/staging/prod, fast rebuilds, and pipelines that run `plan` on each PR (Git as the source of truth, the GitOps idea from Session 14).
- **Analogy: an architect's blueprint.** Any builder can build the identical house from it, any number of times, and every change is drawn on the blueprint first.

| | **Imperative** ("how") | **Declarative** ("what") |
|---|---|---|
| You write | The steps: "create a server, then another, then open port 80" | The end result: "2 servers with port 80 open" |
| The tool | Runs your steps exactly (run it twice: 4 servers!) | Compares what exists with what you asked for, works out the steps; run twice, changes nothing |
| Examples | A shell script, `kubectl create`, `kubectl scale` | **Terraform**, `kubectl apply -f`, CloudFormation |

**Idempotent** ("eye-dem-POH-tent") = doing it many times gives the same result as doing it once: `mkdir -p` and `>` are; `mkdir` (2nd run: "File exists") and `>>` (one more line every run) aren't.
- Terraform with "1 bucket" in the file creates it once, then every later run says **"No changes."**, so IaC is safe to re-run.
- **Mutable** infrastructure (changeable) is patched in place and breeds snowflakes; **immutable** infrastructure is never modified: you build a new server from the code and replace the old one (like a new image replacing Pods).

**Two tool families (exam favourite):** **provisioning** tools create the infrastructure itself (networks, VMs (virtual machines), load balancers, DNS); **configuration management** tools set up software *inside* existing servers (install nginx, edit config, create users).
- **Ansible** is **agentless** (it connects over SSH and runs YAML **playbooks**); **Chef** (Ruby "recipes") and **Puppet** ("manifests") need an **agent**, a small daemon on every server that pulls instructions from a central server.
- Common combo: Terraform builds the servers, then Ansible configures them.

| | **Terraform** (HashiCorp, now IBM) | **Ansible** (Red Hat) | **CloudFormation** (AWS, Amazon Web Services) |
|---|---|---|---|
| Main job · style | **Provisioning** · **declarative** | **Configuration management** · **procedural** (tasks run top to bottom) | **Provisioning** · **declarative** |
| Which clouds | **Any** (via plugins called providers) | Any server it can SSH into | **AWS only** |
| Language | **HCL** (HashiCorp Configuration Language) | **YAML** playbooks | JSON or YAML templates |
| State file? · agent? | **Yes** (Concept 88) · no: calls the cloud's API | **No**: checks live servers each run · no: uses SSH | AWS tracks a "stack" · no: AWS runs it |

### 🛠 Do (2 min): feel idempotency
```bash
mkdir -p ~/devops-practice/iac-demo && cd ~/devops-practice/iac-demo
mkdir data; mkdir data                # 2nd mkdir: "File exists"
mkdir -p data && echo "still fine"    # -p never complains
echo hi >> a.txt; echo hi >> a.txt
echo hi > b.txt; echo hi > b.txt
wc -l a.txt b.txt                     # 2 lines vs 1
```
### ✅ Check 86

**86.1** Which command can be run three times in a row and leave **exactly the same result** as running it once?
- A) `echo "export X=1" >> ~/.bashrc`
- B) `git commit --allow-empty -m run`
- C) `kubectl apply -f deploy.yaml`
- D) `docker run -d --rm nginx:1.27`

**86.2** A team must **(1)** create a VPC, three virtual machines, and a load balancer on AWS, then **(2)** install nginx and copy config files onto those machines. Which split matches the usual tool roles?
- A) Terraform for (1), Ansible for (2)
- B) Ansible for (1), then Terraform for (2)
- C) CloudFormation alone for (1) and (2)
- D) Terraform alone for (1) and (2)

**86.3** (one line) Explain declarative vs imperative in contrast form, with one example of each.

## Concept 87 · Terraform's architecture and the HCL language

**Words you'll learn:** Terraform core, provider, plugin, Terraform Registry, HCL, block, argument, attribute, resource, local name, data source, variable, `.tfvars`, `TF_VAR_`, output, locals, interpolation, dependency graph, `depends_on`, `count`, `for_each`.

**Terraform** (HashiCorp, 2014) is a free command-line tool, `terraform`, that creates, changes, and deletes infrastructure from text files, on almost any cloud.
- **Terraform core** reads your files, works out what depends on what, compares that with what exists, and makes the **plan**; it knows nothing about any cloud.
- **Providers** are **plugins** (add-on programs) that each talk to one platform's **API** (Application Programming Interface): `aws`, `azurerm` (Azure Resource Manager), `google`, `kubernetes`, `helm`, `docker`, `local` (files on your Mac), `random` (random names/passwords).
- The flow: `.tf files → core (graph → compare with state → plan) → providers → each platform's API`.

**Analogy: a travel agent.** You say which trip you want; the agent (core) owns no planes or hotels, but calls each airline's and hotel's own booking system (the providers). The **Terraform Registry** (`registry.terraform.io`) publishes providers and modules, like Docker Hub for Terraform plugins.

**HCL** files end in **`.tf`**; every `.tf` file in a folder is read together, so names and order don't matter.
- Everything is a **block**: `TYPE "LABEL" "LABEL" { argument = value }`, with `"strings"`, numbers, `true`/`false`, lists `["a", "b"]`, maps `{ Name = "web" }`, and `#` comments.
- **Interpolation** puts an expression in a string: `"${var.env}-note"`.

**🛠 Do (5 min): your first configuration.** Run `mkdir -p ~/devops-practice/tf-local && cd ~/devops-practice/tf-local && code main.tf`, paste, save (Practical A runs it):
```hcl
terraform { # settings for Terraform itself: which providers, from where, which versions (~> in ➕)
  required_providers {
    local  = { source = "hashicorp/local", version = "~> 2.5" }
    random = { source = "hashicorp/random", version = "~> 3.6" }
  }
}
variable "greeting" { default = "Hello" } # an INPUT, used as var.greeting
variable "env" { default = "dev" }
resource "random_pet" "name" { length = 2 } # a RESOURCE (something Terraform creates): a 2-word name like "native-worm"
resource "local_file" "note" {
  filename = "${path.module}/output/${var.env}-note.txt" # path.module = this folder
  content  = "${var.greeting}, ${random_pet.name.id}! This file was made by Terraform.\n"
}
output "pet_name" { value = random_pet.name.id } # printed after apply; also `terraform output`
output "file_path" { value = local_file.note.filename }
```
- Other blocks: `provider "aws" { region = "us-east-1" }` (settings for one provider) · `variable "env" { type = string }` (types: string, number, bool, list(string)…)
  - `locals { prefix = "counter-${var.env}" }` (helper values, used as `local.prefix`)
  - `data "aws_availability_zones" "all" {}` (a **data source**: LOOKS UP what exists, creates nothing; used as `data.aws_availability_zones.all.names`).
- In `resource "random_pet" "name"`, **`random_pet`** is the **resource type** (the part before the first `_` names the provider) and **`name`** is the **local name**: your nickname inside the code only. Refer to it as **`TYPE.NAME.ATTRIBUTE`**.
- **Arguments** are what you set; **attributes** are what you read back, including values decided at creation (`id`, `arn`), shown in a plan as **`(known after apply)`**.
- ⚠️ **Traps:** the block is **`locals`** (plural) but you write **`local.prefix`** (singular); a data source needs the **`data.`** prefix.
- **Variable values, weakest → strongest:** `default` < environment variable **`TF_VAR_<name>`** < **`terraform.tfvars`** (auto-loaded) < `*.auto.tfvars` < **`-var 'env=prod'`** / **`-var-file=prod.tfvars`** on the command line.
  - ⚠️ So `terraform.tfvars` **beats** `TF_VAR_`. With no value anywhere, Terraform asks (`var.env  Enter a value:`); with `-input=false` it errors.
- **The dependency graph.** You never write "create the pet first". Because the file *references* `random_pet.name.id` (or a subnet references `aws_vpc.main.id`), there's an **implicit dependency** (implied, not written out).
  - From all references Terraform builds a **dependency graph** (a map of what needs what): independent resources are created **in parallel** (up to 10 at once), and **`destroy` runs in reverse**.
  - A hidden need without a reference is written as **`depends_on = [aws_iam_role_policy.x]`** (rare).
- **Many copies:** `count = 3` makes `web[0]`, `web[1]`, `web[2]` (`count.index`); `for_each = toset(["dev", "prod"])` makes `env["dev"]`, `env["prod"]` (`each.key`). `for_each` is often preferred: removing a middle item from a `count` list shifts the numbers after it, so those get recreated.
### ✅ Check 87

**87.1** A VPC is declared as `resource "aws_vpc" "main" { … }`. Inside `resource "aws_subnet" "public"`, what should `vpc_id =` be set to?
- A) `resource.aws_vpc.main`
- B) `var.aws_vpc.main.id`
- C) `aws_vpc["main"].id`
- D) `aws_vpc.main.id`

**87.2** How does Terraform decide the order in which to create resources?
- A) It follows the order the blocks appear in the files
- B) It builds a graph from references between resources
- C) It creates resources alphabetically by their type
- D) It needs depends_on on every resource that has a parent

**87.3** (one line) What's the difference between a `resource` block and a `data` block?

## Concept 88 · The Terraform workflow and the state file

**Words you'll learn:** working directory, `init`, `.terraform/`, lock file, `validate`, `fmt`, `plan`, dry run, refresh, forces replacement, `apply`, `-auto-approve`, `destroy`, state, `terraform.tfstate`, drift, backend, remote backend, state locking, `force-unlock`, `import`.

**Workflow:** `write .tf → init (once) → fmt / validate → plan → apply → … → destroy`, run in the **working directory** (the folder with your `.tf` files).
| Command | What it does |
|---|---|
| `terraform init` | **Initialise**: downloads providers into a hidden **`.terraform/`** folder, sets up the backend, and writes **`.terraform.lock.hcl`** (exact provider versions + hashes, i.e. fingerprints; **commit it**). Re-run after adding a provider or backend. Forgotten → `Inconsistent dependency lock file … run: terraform init` |
| `terraform fmt` / `validate` | `fmt` rewrites files in the standard layout (**`fmt -check`** only exits **3** if a file is messy: pipelines fail on it). `validate` checks syntax and references without contacting any cloud (needs `init`): `Reference to undeclared resource` or `Success! The configuration is valid.` |
| `terraform plan` | A **dry run** (a rehearsal that changes nothing): **refreshes** (reads what really exists), compares it with your code, prints the **execution plan** |
| `terraform apply` | Makes a fresh plan and asks; only the exact word **`yes`** continues. **`-auto-approve`** skips it (pipelines). `plan -out=tfplan` + `apply tfplan` runs exactly the reviewed plan |
| `terraform destroy` | Deletes **everything in the state**, in **reverse** dependency order, after `yes`; never touches what it didn't create |
| `terraform state list` / `state show ADDR` | List the managed resources / show one's stored attributes. `terraform output -raw NAME` prints one output's value |
| `state mv OLD NEW` / `state rm ADDR` / `import ADDR ID` | Rename an address without destroying it / **forget** a resource (the real thing is **not** deleted) / **adopt** something made by hand |

| Plan symbol (learn cold) | Meaning | Example cause |
|---|---|---|
| **`+`** / **`-`** | **create** / **destroy** | A new resource block / a deleted resource block |
| **`~`** | **update in place** (same object, modified) | Changing a **tag** on a VPC |
| **`-/+`** | **replace: destroy first, then create** | A change marked **`# forces replacement`** (e.g. a server's `ami`) |

The summary reads `Plan: 2 to add, 0 to change, 0 to destroy.`; a replacement counts as **1 to add + 1 to destroy**. ⚠️ `-/+` isn't "maybe": the old object, with its data and IP address, **will be deleted**.

**State: Terraform's memory.** After the first apply, **`terraform.tfstate`** (JSON) records every managed resource, the real-world ID it maps to (`aws_vpc.main` = `vpc-6fdce8…`), and its attributes.
- Terraform needs it to know **what it owns**, to compute the plan (code vs state vs reality), and to delete in the right order.
- **Analogy: a hotel's room register.** The booking says "#42" and the building has 300 rooms; only the register says "#42 = room 305".
- ⚠️ **Rules:** it holds **secrets in plain text** (even `sensitive` values); **never commit it to Git**; **never edit it by hand** (a `terraform.tfstate.backup` is kept).
- **Drift detection:** after a manual change, the next `plan` refreshes, notices, and plans to restore what the code says; `apply` undoes the change, or you update the code to keep it.

**Remote state and locking.** A **backend** = where the state is stored. The default **local backend** (a file in your folder) fails teams: different copies, two applies at once.
- A **remote backend** keeps one shared, locked copy, e.g. inside the `terraform {}` block: `backend "s3" { bucket = "riya-terraform-state"  key = "counter/prod/terraform.tfstate"  region = "us-east-1"  use_lockfile = true }` (each argument on its own line).
  - **S3** = AWS's file storage; `use_lockfile` needs Terraform 1.10+, older setups lock with `dynamodb_table`, **DynamoDB** being AWS's simple database.
- Other backends: **HCP Terraform** (HCP = HashiCorp Cloud Platform), `azurerm`, `gcs`.
- **State locking:** while one apply runs, a second gets `Error: Error acquiring the state lock` with `Lock Info` (ID, Who, Operation). **Wait**; only if a crashed run left a **stale** lock and nothing is running, use `terraform force-unlock <ID>`.

### 🛠 Do: Practical A (pure local, ~15 min; uses Concept 87's `main.tf`)
```bash
cd ~/devops-practice/tf-local
terraform init && terraform validate && terraform plan   # Installed hashicorp/local …, random … Success! … two "+" resources; Plan: 2 to add
terraform apply && cat output/dev-note.txt   # yes → Apply complete! 2 added; Hello, native-worm! This file was made by Terraform.
terraform plan            # No changes. Your infrastructure matches the configuration.  ← idempotent
terraform state list      # local_file.note, random_pet.name
terraform output -raw pet_name; echo     # the name
terraform plan -var 'greeting=Namaste'   # -/+ local_file.note, content # forces replacement; 1 to add, 1 to destroy
printf 'greeting = "Namaste"\nenv      = "prod"\n' > prod.tfvars && terraform apply -var-file=prod.tfvars   # yes → prod-note.txt replaces dev-note.txt
TF_VAR_greeting=Hola terraform plan -var-file=prod.tfvars   # No changes: the file beats TF_VAR_
rm output/prod-note.txt && terraform plan -var-file=prod.tfvars   # drift: "local_file.note will be created"
terraform apply -auto-approve -var-file=prod.tfvars   # the code wins: the file is back
terraform destroy -var-file=prod.tfvars  # yes → the file goes first, then the pet (reverse order)
```
*The `local` provider can't edit a file in place, so a content change **forces replacement**. Single quotes keep `greeting=Namaste` one word (the outer quotes decide). `VAR=value command` sets an env var for that one command only.*
### ✅ Check 88

**88.1** `terraform plan` shows `-/+ resource "aws_instance" "web"` with `ami = "ami-0aaa" -> "ami-0bbb" # forces replacement`. What will `apply` do to the server?
- A) Change the AMI on the running instance in place
- B) Create a second instance and keep both running
- C) Delete the instance, then create a new one
- D) Nothing, since -/+ only marks a dry-run preview

**88.2** A teammate wants to commit `terraform.tfstate` to the team's GitHub repo "so everyone has the latest copy". What's the best response?
- A) No: use a remote backend with locking instead
- B) Fine, as long as the repo is private to the team
- C) Fine, after running `terraform fmt` on the state
- D) No: delete the state and rely on `terraform plan`

**88.3** You apply a security group with one inbound rule. A colleague then adds a second inbound rule **by hand in the AWS console**. What does your next `terraform plan` show?
- A) No changes, since state still matches your code
- B) An update that adds the new rule into your .tf file
- C) An error, because the state file is now corrupted
- D) An update that removes the rule added by hand

## Concept 89 · The cloud, its building blocks, and provisioning with Terraform

**Words you'll learn:** cloud computing, on-premises, pay-as-you-go, elasticity, IaaS / PaaS / SaaS, FaaS, public / private / hybrid / multi-cloud, region, Availability Zone (AZ), high availability, VPC, VNet, subnet, internet gateway (IGW), route table, NAT gateway, security group, NACL, stateful / stateless, EC2, AMI, S3, IAM, role, policy, ARN, least privilege, EKS / AKS / GKE.

**The problem.** **On-premises** ("on-prem": servers in your own building) means buying servers months ahead, guessing how many, and waiting weeks for them.
- **Cloud computing** = renting someone else's computers, storage, and networks over the internet, on demand, paying only for what you use (**pay-as-you-go**, per second or hour).
- **Elasticity** = growing and shrinking automatically with demand (the HPA idea, for whole servers). The big three: **AWS**, **Microsoft Azure**, and **GCP** (Google Cloud Platform).

**Service models: who manages what** (the "Pizza as a Service" analogy; on-prem = homemade, you do everything):
| Model | You manage | Provider manages | Examples | Pizza version |
|---|---|---|---|---|
| **IaaS** (Infrastructure as a Service) | OS (operating system), runtime, app, data | Hardware, network, virtualisation | **AWS EC2**, Azure VMs, Compute Engine | Take-and-bake: you bake it at home |
| **PaaS** (Platform as a Service) | Only your app and data | OS, runtime, scaling | **Heroku**, App Engine, Elastic Beanstalk | Delivery: you just set the table |
| **SaaS** (Software as a Service) | Just your data | Everything | **Gmail**, Salesforce, Microsoft 365 | Dining out |

- **In between:** **CaaS** (Containers as a Service, e.g. managed Kubernetes) and **serverless / FaaS** (Function as a Service, e.g. **AWS Lambda**: one function, run only when called, paid per call).
- **Deployment models:** **public** cloud (shared, rented by anyone) · **private** cloud (cloud-style, one organisation only, e.g. OpenStack in a bank) · **hybrid** (on-prem + public, connected) · **multi-cloud** (several public providers; one Terraform for all).
- **Regions and AZs:** a **region** = a separate geographic area with data centres (`us-east-1` N. Virginia, `ap-south-1` Mumbai). An **Availability Zone (AZ)** = one or more data centres inside a region, with their own power, cooling, and network, linked to the others by fast connections (`ap-south-1a`, `ap-south-1b`).
- **High availability (HA)** = the app keeps working when a part fails: run servers in 2+ AZs behind a load balancer, so a fire in one AZ doesn't take you down. Choose a region by **latency** (delay to users), data laws, price, and available services.
#### Networking (the heart of the lecture)
- **VPC** (**Virtual Private Cloud**; Azure: **VNet**, Virtual Network) = your own isolated private network in the cloud, with a **CIDR block** such as `10.0.0.0/16` (65,536 private addresses), in one region, spanning all its AZs.
- **Subnet** = a smaller range carved from the VPC, in **exactly one AZ**: `10.0.1.0/24` (256; AWS reserves 5, so 251 usable). A **public subnet**'s route table sends `0.0.0.0/0` to an internet gateway; a **private subnet**'s doesn't (databases).
- **Internet gateway (IGW)** = the VPC's door to the internet: one per VPC, useless until a route points to it.
- **Route table** = rules "destination CIDR → target": `10.0.0.0/16 → local` (automatic) and `0.0.0.0/0 → igw-…` (everything else). The **most specific** match wins.
- **NAT gateway** (Network Address Translation), placed in a public subnet, lets private servers start connections **out** (updates) while nothing outside can start one **in**.
- ⚠️ **Trap:** public vs private is decided by the **route table**, not by the CIDR range (both use 10.x).
```
 ┌ VPC 10.0.0.0/16 ──────────── Internet Gateway ⇄ INTERNET ──────────────────────────────────┐
 │ Public subnet 10.0.1.0/24 (AZ a)              │ Private subnet 10.0.2.0/24 (AZ b)           │
 │ routes: 10.0.0.0/16 → local, 0.0.0.0/0 → IGW  │ routes: 10.0.0.0/16 → local, 0.0.0.0/0 → NAT │
 │ [EC2 web] ◄ SG: 22, 80    [NAT gateway] ──────┼──► [database] ◄ SG: 5432 from web SG only    │
 └────────────────────────────────────────────────────────────────────────────────────────────┘
```
**Two firewalls.** A **security group (SG)** is a firewall on an instance; a **NACL** (**Network Access Control List**) is a firewall on a whole subnet. **Ephemeral ports** (1024–65535) are the temporary ports a client picks for its side of a connection.
| | **Security group** | **NACL** |
|---|---|---|
| Protects | **An instance** (your main firewall) | **A whole subnet** (a coarse extra layer) |
| Rules | **Allow only** (new SG: no inbound, all outbound); can allow "from another SG" | **Allow and deny**, numbered (default NACL allows all) |
| Memory | **Stateful**: replies are allowed automatically | **Stateless**: replies need their own rule (ephemeral ports) |
| Evaluation | All rules considered together | Lowest number first; first match wins |

- **EC2** (**Elastic Compute Cloud**) = AWS's virtual machines; one VM = an **instance**; **instance type** = its size (`t3.micro`); **AMI** (**Amazon Machine Image**) = the template disk it boots from; **key pair** = SSH keys (you keep the private one); **user data** = a script run once at first boot.
- **S3** (**Simple Storage Service**) = object storage: **buckets** (names unique across all of AWS) hold **objects** (files), read and written over HTTP (backups, static sites, Terraform state).
- **IAM** (**Identity and Access Management**) = who can do what:
  - a **user** (long-term credentials), a **group** (users sharing permissions), a **role** (permissions **assumed temporarily** by a service, instance, or pipeline; no stored keys, so preferred),
  - and a **policy** (a JSON permission list, e.g. `{"Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::counter-assets/*"}`). An **ARN** (**Amazon Resource Name**) is a resource's unique ID.
  - Everything is denied by default; an explicit Deny beats an Allow. **Least privilege** = only the permissions needed. Lock the **root user** away with **MFA** (Multi-Factor Authentication).

### 🛠 Terraform in action: VPC + subnet + IGW + route table + SG + EC2 (Practical B, ~15 min)
**Practical B** builds the network for real against **LocalStack** (AWS's APIs imitated at `http://127.0.0.1:4566`), using the real `aws` provider with **fake credentials**: nothing reaches Amazon or costs money.
- ⚠️ The `aws` provider is a ~770 MB download, so the first `init` takes minutes.
- Run `mkdir -p ~/devops-practice/tf-cloud && cd ~/devops-practice/tf-cloud && code main.tf`, then paste:
```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test" # fake keys: LocalStack accepts anything.
  secret_key                  = "test" # NEVER write real keys in a .tf file
  skip_credentials_validation = true   # don't ask real AWS whether the keys are valid
  skip_requesting_account_id  = true   # don't look up a real AWS account number
  skip_metadata_api_check     = true   # don't look for an AWS server's built-in credentials
  s3_use_path_style           = true   # use 127.0.0.1:4566/bucket, not bucket.127.0.0.1
  endpoints {                          # send these services' API calls to LocalStack
    s3  = "http://127.0.0.1:4566"
    ec2 = "http://127.0.0.1:4566" # EC2's API also handles VPCs, subnets, and SGs
    sts = "http://127.0.0.1:4566" # STS = Security Token Service, AWS's "who am I"
  }
}
resource "aws_s3_bucket" "files" { bucket = "practice-bucket" }
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "practice-vpc" }
}
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id # reference → built after the VPC
  cidr_block              = "10.0.1.0/24"   # a /24 slice of the VPC, in ONE AZ
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true # instances here get a public IP
}
resource "aws_internet_gateway" "igw" { vpc_id = aws_vpc.main.id } # the VPC's door
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"                 # "everything else"...
    gateway_id = aws_internet_gateway.igw.id # ...goes out through the IGW
  }
}
resource "aws_route_table_association" "public" { # THIS makes the subnet public
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
  ingress { # HTTP in from anyone (SSH would be port 22 from ["YOUR.IP/32"]: /32 = one address)
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress { # all out: Terraform removes AWS's default outbound rule unless you write one
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # -1 = all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }
}
data "aws_availability_zones" "all" { state = "available" } # a data source: reads, creates nothing
output "azs" { value = data.aws_availability_zones.all.names }
```
```bash
terraform init && terraform validate && terraform apply   # Installed hashicorp/aws v6.x … Success! … Plan: 7 to add → type yes
curl -s http://127.0.0.1:4566/practice-bucket   # XML containing <Name>practice-bucket</Name>; the bucket exists
```
*The VPC and the bucket are created together (neither references anything); the rest wait for the VPC. The output `azs` lists `us-east-1a` … `us-east-1f`.*
- *Add `Env = "dev"` to the VPC's `tags` and `terraform plan` shows `~ update in-place`, `0 to add, 1 to change`.*
- ***Clean up:** `terraform destroy` (the VPC goes **last**), then `rm -rf .terraform` (frees ~770 MB).*

**The server, on paper only** (LocalStack's free edition can't boot a real VM):
```hcl
resource "aws_instance" "web" {
  ami                    = "ami-0abc123example" # an Ubuntu AMI ID (a data "aws_ami" block can look it up)
  instance_type          = "t3.micro"           # 2 virtual CPUs, 1 GB RAM
  subnet_id              = aws_subnet.public.id # in the public subnet
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = "vansh-laptop" # an existing key pair, for SSH
  user_data              = "#!/bin/bash\napt-get update -y && apt-get install -y nginx"
}
```
After apply, an output of `aws_instance.web.public_ip` (known after apply) prints e.g. `54.x.x.x`.
- **Request path (draw it in exams):** browser → public IP → **IGW** → route table (`0.0.0.0/0 → IGW`) → public subnet → **SG** (port 80 allowed?) → EC2 nginx; the stateful SG lets the reply out.
- **If it times out, check in order:** public IP? route to the IGW? SG port? app listening on `0.0.0.0` (not `127.0.0.1`)?

**Managed Kubernetes (awareness):** **EKS** (Amazon Elastic Kubernetes Service), **AKS** (Azure Kubernetes Service), and **GKE** (Google Kubernetes Engine) run the control plane for you; you pay for worker nodes in your subnets, and `type: LoadBalancer` gets a real cloud load balancer. Typical: Terraform builds the VPC + EKS, then Helm or Argo CD deploys the apps (Session 16). **Same idea, other clouds:** Azure = Virtual Machines, Blob Storage, **VNet**, NSG (Network Security Group), AKS, ARM (Azure Resource Manager) templates / Bicep; Google = Compute Engine, Cloud Storage, VPC, firewall rules, GKE.
### ✅ Check 89

**89.1** A startup pushes its code to a platform that builds and runs it. They never patch an operating system or manage servers, but they do write and maintain the app. Which model is this?
- A) IaaS, like renting EC2 virtual machines
- B) SaaS, like Gmail or Salesforce for users
- C) PaaS, like Heroku or Elastic Beanstalk
- D) On-premises, like servers in your office

**89.2** A security group allows **inbound TCP 80 from 0.0.0.0/0** and has **no outbound rules at all**. A user's browser requests a page from the instance. What happens to the response?
- A) It's blocked, as there's no outbound rule for it
- B) It's allowed, as security groups are stateful
- C) It's allowed only if the reply also uses port 80
- D) It's blocked unless the instance uses a NAT gateway

**89.3** (one line) Explain region vs Availability Zone in contrast form, and why you'd spread servers across two AZs.

## ➕ Small but important (Terraform & Cloud): read before the quiz
| Item | What it is |
|---|---|
| **Workspaces** | Several **separate state files from the same code**: `terraform workspace new dev / select prod / list / show`; every folder starts in `default`; code reads `terraform.workspace` |
| **Modules** | A **folder of `.tf` files reused like a function**; yours is the **root module**. `module "vpc" { source = "terraform-aws-modules/vpc/aws" … }`, `init`, then `module.vpc.vpc_id` |
| **`sensitive = true`** | Shows `(sensitive value)` on screen, but ⚠️ it's still **plain text in the state**. An output using a sensitive variable must also say `sensitive = true` |
| **`lifecycle` block** | **`prevent_destroy = true`**: any plan that would delete it fails (databases). **`create_before_destroy = true`**: new one first (`+/-`). **`ignore_changes = [tags]`** |
| **Version constraints** | `"~> 6.0"` = **≥ 6.0 and < 7.0**; `"~> 6.4.0"` = ≥ 6.4.0 and < 6.5.0; `">= 1.5"` = that or newer; `"= 2.1.0"` = exactly. `~>` = the "pessimistic constraint" |
| **`.gitignore`** · credentials | Ignore `.terraform/`, `*.tfstate`, `*.tfstate.*`, `crash.log`, secret `*.tfvars`; **commit** `.tf` files and **`.terraform.lock.hcl`**. Cloud keys never go in `.tf` files: use `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, a profile, or in GitHub Actions an **OIDC** (OpenID Connect) role with short-lived credentials |
| **OpenTofu** | 2023: Terraform's licence moved from open source (MPL, Mozilla Public License) to **BSL** (Business Source License); the community **forked** (copied) it as OpenTofu: `tofu init/plan/apply` |
| **Infracost / tfsec / Checkov** | Pipeline tools: Infracost posts a change's **monthly cost** on the PR; tfsec (now **Trivy**: `trivy config .`) and Checkov **scan IaC** (public buckets, port 22 open to the world) |
| **Shared responsibility** | The provider secures **"OF the cloud"** (buildings, hardware, hypervisors); **you** secure **"IN the cloud"** (data, IAM, SG rules, encryption, OS patches on IaaS) |
| `-target=` · `-replace=` · `TF_LOG=DEBUG` | Plan/apply one resource only (emergencies) · force one to be recreated (replaces `terraform taint`) · very detailed debug logs. **Provisioners** (`local-exec`/`remote-exec`) = a last resort |
| **CIDR quick maths** | `/16` = 65,536 · `/24` = 256 (**251** usable in AWS) · `/32` = **one** address · `0.0.0.0/0` = **every** address |

**Quick checks:** (1) You mark `variable "db_password"` as `sensitive = true`. Is the password safe to commit inside `terraform.tfstate`? (2) What does `version = "~> 5.2"` allow?
*Answers:* (1) No. `sensitive` only hides it from the screen; the state stores it in plain text, so the state must never go into Git (use an encrypted remote backend). (2) Any version from 5.2 up to, but not including, 6.0 (5.2, 5.3, … 5.99).

## 📝 Session quiz (4 MCQs + 2 one-liners), from memory

**Q1.** After a clean apply, you change **two things**: (1) you add a tag to `aws_vpc.main`, and (2) you change the `ami` of `aws_instance.web` (which forces replacement). Which summary line does `terraform plan` print?
- A) `Plan: 0 to add, 2 to change, 0 to destroy.`
- B) `Plan: 1 to add, 1 to change, 1 to destroy.`
- C) `Plan: 2 to add, 0 to change, 2 to destroy.`
- D) `Plan: 1 to add, 2 to change, 1 to destroy.`

**Q2.** Two engineers work on the same Terraform code from Git, and **each keeps `terraform.tfstate` on their own laptop**. Engineer A runs `apply` and creates a VPC. Engineer B, with an up-to-date checkout, has never applied. B runs `terraform plan`. What does B see?
- A) No changes, because the code in Git is identical
- B) A lock error, because A's state is still locked
- C) An error saying the VPC is owned by engineer A
- D) A plan to create the VPC, as B's state is empty

**Q3.** `variable "env"` has `default = "dev"`. The folder contains `terraform.tfvars` with `env = "staging"`. You run `export TF_VAR_env=qa`, then `terraform apply` with **no** `-var` flags. Which value does `var.env` get?
- A) `dev`, as the default in the variable block wins
- B) `qa`, as environment variables override files
- C) `staging`, as terraform.tfvars beats TF_VAR_
- D) It asks, because three values are in conflict

**Q4.** An EC2 instance in subnet `10.0.1.0/24` has a **public IP**. Its security group allows TCP 80 from `0.0.0.0/0`, and nginx is running and listening on `0.0.0.0:80`. The subnet's route table contains **only** `10.0.0.0/16 → local`. A browser request to the public IP **times out**. What's needed?
- A) Route 0.0.0.0/0 to an internet gateway
- B) An inbound NACL rule that allows port 80
- C) An outbound security group rule for port 80
- D) Move the instance to a subnet in another AZ

**L1.** Write the full forms of **IaC**, **HCL**, **VPC**, and **IAM**, and name the two things `terraform init` creates in your folder.

**L2.** Explain **security group vs NACL** in contrast form, giving two differences.
