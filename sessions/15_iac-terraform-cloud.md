# Session 15 · Infrastructure as Code, Terraform & Cloud (Oct 18, Session 1)

**Time:** ~120–140 min (two lectures in one) · **Concepts:** 86 → 89 · **Answers:** `MY_ANSWERS.md` or say "Terraform questions" · **Answer key:** `15_iac-terraform-cloud_ANSWERS.md`

**How to read this:** this session covers two lectures ("Terraform & IaC" and "Cloud & Terraform in Action"). New words are **bold** and explained where they first appear. You need **no cloud account**: Practical A runs only on your Mac, and Practical B uses **LocalStack** (a pretend Amazon cloud that's already running on your Mac).

**Before you start:** `terraform version` should print `Terraform v1.16.x on darwin_arm64` (an "out of date!" note is harmless), and `docker ps --filter name=localstack --format '{{.Names}}  {{.Status}}'` should print `localstack  Up … (healthy)` (if Colima is stopped, run `colima start` first).
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

### The problem
**Infrastructure** = the servers, networks, firewalls, databases, load balancers and DNS records an app runs on. To **provision** = to create and set up infrastructure.

Riya's team moves the counter app to Amazon's cloud. Asha builds everything by clicking in the **cloud console** (the cloud company's website) and doesn't write the steps down. Building infrastructure by clicking like this is nicknamed **ClickOps**. Six months later nobody can build an identical test copy (nobody remembers the exact clicks), a port someone opened by hand during a 2 a.m. outage is still open, and nobody can say who changed what, or undo it. Two names for where this ends up:
- A **snowflake server** has been patched by hand so many times that it's **unique**, like a snowflake. Nobody can rebuild it, so everyone is scared to touch it.
- **Configuration drift** = the real setup slowly **drifts away** from what everyone thinks it is, through small manual changes.

### The idea: Infrastructure as Code
> **Infrastructure as Code (IaC)** means describing your infrastructure in **text files**, and letting a **tool create, change and delete the real infrastructure to match those files**.

**Analogy: an architect's blueprint.** Any builder can build the identical house from it, any number of times, and every change is drawn on the blueprint first. Because the files live in **Git**, you get history, review of every change in a pull request, identical dev/staging/prod built from the same code, fast rebuilds, and pipelines that check each change (Git as the single source of truth: the GitOps idea, where an agent keeps the real system matching what's in Git).

### Declarative vs imperative
| | **Imperative** ("how") | **Declarative** ("what") |
|---|---|---|
| You write | **The steps**: "create a server, then another, then open port 80" | **The end result**: "there should be 2 servers with port 80 open" |
| The tool | Runs your steps exactly | Compares what exists with what you asked for, and **works out the steps itself** |
| Run it twice | Does the work twice (4 servers!) | Sees it's already done and changes nothing |
| Examples | A shell script, `kubectl create`, `kubectl scale` | **Terraform**, `kubectl apply -f`, CloudFormation |

**Analogy: a taxi.** Imperative is telling the driver "left, right, second right…"; declarative is saying "to the airport, please" and letting the driver work out the route. Kubernetes already works declaratively; Terraform does the same for cloud infrastructure.

### Idempotent: safe to run again
> An action is **idempotent** ("eye-dem-POH-tent") when **doing it many times gives the same result as doing it once**.

**Analogy:** a lift call button. Press it once or five times, and **one** lift comes. `mkdir -p data` and `echo hi > b.txt` are idempotent; `mkdir data` (the 2nd run fails: "File exists") and `echo hi >> a.txt` (one more line every run) aren't. Terraform with "1 bucket" in the file creates it on the first run, then says **"No changes."** every time after. That's why IaC is safe to re-run, and why pipelines can run it automatically. A related pair of words:
- **Mutable** infrastructure (mutable = changeable): you **change servers in place** (log in, upgrade, edit configs). Over time each one becomes a snowflake.
- **Immutable** infrastructure (immutable = unchangeable): you **never modify** a running server. To change something, you **build a new one from the code and replace the old one**, just as Kubernetes replaces Pods with ones from a new image.

### Two jobs: provisioning vs configuration management (exam favourite)
**Provisioning** tools **create the infrastructure itself** (networks, VMs (virtual machines), load balancers, DNS): building the house. **Configuration management** tools **set up the software inside servers that already exist** (install nginx, edit config files, create users): furnishing the rooms. They're often **combined**: Terraform creates the servers, then Ansible configures them.
- **Ansible** connects to servers over **SSH** and runs tasks from YAML files called **playbooks**. It's **agentless**: nothing has to be installed on the servers first.
- **Chef** and **Puppet** need an **agent**: a small background program (a daemon) on every server that regularly pulls its instructions from a central server. Chef's instructions are "recipes" (in Ruby); Puppet's are "manifests".

| | **Terraform** (HashiCorp, now part of IBM) | **Ansible** (Red Hat) | **CloudFormation** (AWS, Amazon Web Services) |
|---|---|---|---|
| Main job · style | **Provisioning** · **declarative** | **Configuration management** · **procedural** (tasks run top to bottom) | **Provisioning** · **declarative** |
| Which clouds | **Any** (through plugins called providers) | Any server it can SSH into | **AWS only** |
| Language | **HCL** (HashiCorp Configuration Language) | **YAML** playbooks | JSON or YAML templates |
| State file? · agent? | **Yes** (`terraform.tfstate`, its record of what it built) · no: calls the cloud's API | **No**: checks live servers each run · no: SSH | AWS tracks it (a "stack") · no |

### 🛠 Do (2 min): feel idempotency
```bash
mkdir -p ~/devops-practice/iac-demo && cd ~/devops-practice/iac-demo
mkdir data; mkdir data                # the 2nd one fails: "mkdir: data: File exists"
mkdir -p data && echo "still fine"    # -p never complains
echo hi >> a.txt; echo hi >> a.txt; echo hi > b.txt; echo hi > b.txt
wc -l a.txt b.txt                     # 2 a.txt (>> appends every time), 1 b.txt (> replaces)
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

### What Terraform is, and how it's built
**Terraform** is a free command-line tool, `terraform`, that **creates, changes and deletes infrastructure from text files**, on almost any cloud. It's split into two kinds of program:
1. **Terraform core** is the `terraform` program itself. It **reads your files**, works out **what depends on what**, **compares** what you asked for with what exists, and makes the **plan**. It knows nothing about any cloud on its own.
2. **Providers** are **plugins** (add-on programs) that each know how to talk to **one platform's API** (Application Programming Interface: the way programs ask a service to do things). Core says "create a bucket"; the `aws` provider makes the right AWS API calls. Others: `azurerm` (Microsoft Azure), `google`, `kubernetes`, `helm`, `docker`, and the two Practical A uses: `local` (**files on your Mac**) and `random` (random names and passwords).

The flow: `your .tf files → Terraform core (graph → compare with state → plan) → providers → each platform's API`. **Analogy: a travel agent.** You say which trip you want. The agent (core) owns no planes or hotels; it calls each airline's and hotel's own booking system (the providers). The **Terraform Registry** (`registry.terraform.io`) is where providers and reusable modules are published, like Docker Hub for Terraform plugins: `source = "hashicorp/aws"` means "download it from `registry.terraform.io/hashicorp/aws`".

### HCL basics
Terraform files are written in **HCL** (**HashiCorp Configuration Language**) and end in **`.tf`**.
- **Every `.tf` file in a folder is read together**, as if it were one big file, so file names and block order don't matter.
- Everything is a **block**: `BLOCK_TYPE "LABEL" "LABEL" { argument = value }`. Values: `"strings"`, numbers `3`, `true`/`false`, lists `["a", "b"]`, maps `{ Name = "web" }`. Comments start with `#`.
- **Interpolation** puts an expression inside a string with `${ }`: `"${var.env}-note"` becomes `dev-note`.

### 🛠 Do (5 min): your first configuration
Run `mkdir -p ~/devops-practice/tf-local && cd ~/devops-practice/tf-local && code main.tf`, then paste this and save (Practical A runs it):
```hcl
terraform { # settings for Terraform itself: which providers to download, from where, which versions
  required_providers {
    local  = { source = "hashicorp/local", version = "~> 2.5" }  # makes files on your Mac (~> is in ➕)
    random = { source = "hashicorp/random", version = "~> 3.6" } # makes random values
  }
}
variable "greeting" { # an INPUT, used as var.greeting
  default = "Hello"   # used when nobody gives a value
}
variable "env" {
  default = "dev"
}
resource "random_pet" "name" { # a RESOURCE (something Terraform creates)
  length = 2                   # a random two-word name, like "native-worm"
}
resource "local_file" "note" {
  filename = "${path.module}/output/${var.env}-note.txt" # path.module = this folder
  content  = "${var.greeting}, ${random_pet.name.id}! This file was made by Terraform.\n"
}
output "pet_name" { # printed after apply, and by `terraform output`
  value = random_pet.name.id
}
output "file_path" {
  value = local_file.note.filename
}
```

### The blocks, explained
- **`terraform`**: settings for Terraform itself, mainly **which providers** to download, from where, and which versions.
- **`variable`**: an **input**, used as **`var.NAME`**, so the same files can build dev, staging and prod with different values. An optional `type` can be string, number, bool, list(string)…
- **`output`**: a value **printed after `apply`** (and any time with `terraform output`), like a receipt. It's how you hand results, such as an IP address, to people or scripts.
- **`resource`**: something Terraform **creates and manages** (the most important block). In `resource "random_pet" "name"`:
  - **`random_pet`** is the **resource type**: the part before the first `_` names the provider (`random`); the rest is the kind of thing.
  - **`name`** is the **local name**: your nickname for it **inside this code only**, not its real name in the cloud. Refer to it elsewhere as **`TYPE.LOCAL_NAME.ATTRIBUTE`**, e.g. `random_pet.name.id`.
  - **Arguments** are what you **set** (`length = 2`). **Attributes** are what you can **read back**, including values the platform decides at creation (an `id`, an IP address); a plan shows those as **`(known after apply)`**.
- **`provider "aws" { region = "us-east-1" }`**: settings for one provider.
- **`locals { prefix = "counter-${var.env}" }`**: named helper values, so you don't repeat an expression. ⚠️ The block is **`locals`** (plural), but you write **`local.prefix`** (singular).
- **`data "aws_availability_zones" "all" {}`**: a **data source**. It **looks up** something that already exists and **creates nothing**. ⚠️ Its reference needs the **`data.`** prefix: `data.aws_availability_zones.all.names`. **Analogy:** a `resource` is **placing an order**; a `data` source is **looking something up in the catalogue**.

**Giving variables a value**, from weakest to strongest (a stronger one wins): the `default` → an environment variable **`TF_VAR_<name>`** (`export TF_VAR_env=qa`) → a **`terraform.tfvars`** file (loaded automatically) → any `*.auto.tfvars` file → the **command line**: `-var 'env=prod'` or `-var-file=prod.tfvars` (strongest).
- ⚠️ **Trap:** `TF_VAR_` is the weakest after the default, so a `terraform.tfvars` file **beats** it.
- With no value anywhere, Terraform stops and asks (`var.env  Enter a value:`); with `-input=false` (pipelines) it errors instead.

### The dependency graph: who gets built first?
You never write "create the pet name first". Terraform works it out from **references**: the file's `content` uses `random_pet.name.id`, so the pet must exist first. A dependency implied by a reference is an **implicit dependency** (implicit = implied, not written out).
- From all the references, Terraform builds a **dependency graph**: a map of what needs what. Things that don't depend on each other are created **at the same time** (in parallel), and **`destroy` runs in reverse** (the file goes before the pet name).
- Rarely, one thing needs another **without referencing it**. Then you write the link out: **`depends_on = [aws_iam_role_policy.x]`**.

**Many copies (awareness):** **`count = 3`** makes `web[0]`, `web[1]`, `web[2]` (inside, `count.index` is 0, 1, 2); **`for_each = toset(["dev", "prod"])`** makes `env["dev"]` and `env["prod"]` (inside, `each.key`). `for_each` is often preferred: with `count`, removing an item from the **middle** of a list shifts the numbers of everything after it, so those get destroyed and recreated.
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

### The workflow
**Workflow:** `write .tf → init (once) → fmt / validate → plan → apply → … → destroy`, run in the **working directory** (the folder with your `.tf` files).
**Analogy: a builder's quote.** `plan` is the builder handing you a written quote ("I'll build 2 walls and knock down 1"); nothing has been touched yet. `apply` is you signing it, and the work starts.

| Command | What it does |
|---|---|
| `terraform init` | **Initialise** the folder (once, and again after adding a provider or backend): download the providers into a hidden **`.terraform/`** folder, set up the backend (where the state lives, below), and write the **lock file `.terraform.lock.hcl`**. The lock file records the **exact provider versions** plus their **hashes** (fingerprints proving the download wasn't tampered with), so everyone gets the same versions: **commit it**. Forget `init`, and `plan` fails with `Inconsistent dependency lock file … run: terraform init` |
| `terraform fmt` | Rewrite the files in the standard layout. **`fmt -check`** changes nothing but exits with code **3** if a file is messy, so a pipeline can fail on it |
| `terraform validate` | Check the syntax, and that every reference points to something that exists, **without contacting any cloud** (needs `init`): `Reference to undeclared resource` or `Success! The configuration is valid.` |
| `terraform plan` | A **dry run** (a rehearsal that changes nothing): it **refreshes** (reads what really exists now), compares that with your code, and prints the **execution plan**: exactly what `apply` would do |
| `terraform apply` | Makes a fresh plan, shows it and asks; only the exact word **`yes`** continues. **`-auto-approve`** skips the question (pipelines, where nobody can type yes). **Saved plans:** `plan -out=tfplan` saves the plan to a file and `apply tfplan` runs **exactly** that plan, so the plan that was reviewed is the plan that runs |
| `terraform destroy` | Deletes **everything recorded in the state**, in **reverse dependency order**, after `yes`. It never touches anything made by hand or by other Terraform folders |

### Reading a plan (learn these symbols cold)

| Symbol | Meaning | Example cause |
|---|---|---|
| **`+`** / **`-`** | **create** / **destroy** | A new resource block / a deleted resource block |
| **`~`** | **update in place**: the same object is modified | Changing a **tag** on a VPC |
| **`-/+`** | **replace**: destroy the old one first, then create a new one | A change that can't be made to a live object, marked **`# forces replacement`** (e.g. a server's `ami`) |

The last line sums it up: `Plan: 2 to add, 0 to change, 0 to destroy.` A replacement counts as **1 to add + 1 to destroy**. ⚠️ **Trap:** `-/+` doesn't mean "maybe": the old object **will be deleted**, and its data and IP address go with it.

### State: Terraform's memory
After the first `apply`, a file called **`terraform.tfstate`** appears. It's JSON, and it records every resource Terraform manages, **the real-world ID it maps to** (`aws_vpc.main` = `vpc-6fdce8…`), and all its attributes. Terraform needs it to know **what it owns** (the cloud may have 12 VPCs; the state says which one is `aws_vpc.main`), to work out the plan (code vs state vs reality), and to delete things in the right order. **Analogy: a hotel's room register.** The booking says "#42" and the building has 300 rooms; only the register says "#42 = room 305". Lose it, and nobody knows which room is whose.

⚠️ **Three rules:** (1) it holds **secrets in plain text** (passwords, keys), even values marked `sensitive`; (2) **never commit it to Git**: it's secret, and two people with two copies cause chaos; (3) **never edit it by hand**: use the `terraform state` commands below (Terraform keeps the previous version as `terraform.tfstate.backup`).

**Drift detection.** If someone changes infrastructure by hand, the next `plan` refreshes, notices the difference, and plans to put things back the way the code says. You then either `apply` (undo the manual change) or update the code (keep it). That's how IaC fights configuration drift.

### Remote state and locking: how teams work
A **backend** is **where Terraform stores the state**. The default **local backend** (a file in your folder) fails a team: everyone has a different copy, and two people can apply at once. Teams use a **remote backend**: one shared copy in central storage.
- The usual AWS one is set inside the `terraform {}` block as `backend "s3" { … }`, an **S3** bucket (AWS's file storage), with a `bucket` (the bucket that holds the state, e.g. `riya-terraform-state`), a `key` (the file's path inside it, e.g. `counter/prod/terraform.tfstate`), a `region`, and `use_lockfile = true` (lock the state while a run is going). Each argument goes on its own line.
- Older setups lock with a **DynamoDB** table (AWS's simple database) instead. Other backends: **HCP Terraform** (HashiCorp's hosted service), `azurerm`, `gcs`.

**State locking:** while one apply runs, the state is **locked**, so a second run has to wait instead of corrupting it (like the "occupied" sign on a single bathroom). The second run gets `Error: Error acquiring the state lock`, with `Lock Info` showing the lock's ID, **Who** holds it, and the Operation. Normally you just **wait**. Only if a crashed run left a **stale** lock behind, and you're sure nothing is running, remove it with `terraform force-unlock <ID>`.

| State command | What it does |
|---|---|
| `terraform state list` / `state show ADDRESS` | List every resource address in the state / show one resource's stored attributes |
| `terraform output -raw NAME` | Print just one output's value, for scripts |
| `terraform state mv OLD NEW` | Rename an address without destroying the resource |
| `terraform state rm ADDRESS` | Make Terraform **forget** a resource. The real thing is **not** deleted |
| `terraform import ADDRESS ID` | **Adopt** something created by hand, so Terraform manages it from now on |

### 🛠 Do: Practical A (pure local, ~15 min; uses Concept 87's `main.tf`)
```bash
cd ~/devops-practice/tf-local
terraform init && terraform validate && terraform plan   # Installed hashicorp/local …, random … Success! … two "+" resources; Plan: 2 to add
terraform apply && cat output/dev-note.txt   # yes → Apply complete! 2 added; Hello, native-worm! This file was made by Terraform.
terraform plan            # No changes. Your infrastructure matches the configuration.  ← idempotent
terraform state list      # local_file.note, random_pet.name
terraform output -raw pet_name; echo     # just the name (echo adds a newline)
terraform plan -var 'greeting=Namaste'   # -/+ local_file.note, content # forces replacement; 1 to add, 1 to destroy
printf 'greeting = "Namaste"\nenv      = "prod"\n' > prod.tfvars && terraform apply -var-file=prod.tfvars   # yes → prod-note.txt replaces dev-note.txt
TF_VAR_greeting=Hola terraform plan -var-file=prod.tfvars   # No changes: the file beats TF_VAR_
rm output/prod-note.txt && terraform plan -var-file=prod.tfvars   # drift: "local_file.note will be created"
terraform apply -auto-approve -var-file=prod.tfvars   # the code wins: the file is back
terraform destroy -var-file=prod.tfvars  # yes → the file goes first, then the pet (reverse order)
```
*Your pet name is random, so it differs from `native-worm`. The `local` provider can't edit a file in place, so a content change **forces replacement**. Single quotes keep `greeting=Namaste` as one word. `VAR=value command` sets an environment variable for that one command only.*
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

### What the cloud is
Before the cloud, companies ran their own servers **on-premises** ("on-prem": in their own building). They had to buy servers months ahead, **guess** how many (too few and the site crashes; too many and money is wasted), and wait weeks for delivery.
> **Cloud computing** = **renting someone else's computers, storage and networks over the internet, on demand, and paying only for what you use.**

**Analogy: electricity.** You don't build a power station; you plug in and pay for the units you use. The big three providers are **AWS** (Amazon Web Services), **Microsoft Azure** and **GCP** (Google Cloud Platform).
- **Pay-as-you-go:** billed per second or hour of use, with no big purchase up front. Turn it off and the bill stops.
- **Elasticity:** automatically **grow and shrink** with demand: more servers during a sale, fewer at night (the HPA idea, for whole servers).

### IaaS, PaaS, SaaS: how much the provider does for you
A running app needs layers: hardware and network → operating system → runtime (e.g. Python) → your app → your data. The **service models** differ in **who manages which layers** (the "Pizza as a Service" analogy; on-prem = homemade, you do everything):

| Model | You manage | The provider manages | Examples | Pizza version |
|---|---|---|---|---|
| **IaaS** (Infrastructure as a Service) | The OS (operating system), runtime, app and data | Hardware, network, virtualisation | **AWS EC2**, Azure VMs, Google Compute Engine | Take-and-bake: you bake it in your own oven |
| **PaaS** (Platform as a Service) | Only your app and data | OS, runtime, scaling | **Heroku**, Google App Engine, AWS Elastic Beanstalk | Delivery: you just set the table |
| **SaaS** (Software as a Service) | Just your data | Everything | **Gmail**, Salesforce, Microsoft 365 | Dining out |

- **In between (awareness):** **CaaS** (Containers as a Service, e.g. managed Kubernetes: you bring containers, the provider runs the cluster) and **serverless** or **FaaS** (Function as a Service, e.g. **AWS Lambda**: you upload one function, it runs only when called, and you pay per call).
- **Deployment models:** a **public** cloud is the provider's shared data centres, rented by anyone (AWS, Azure, GCP); a **private** cloud is cloud-style self-service for **one organisation only** (e.g. a bank running OpenStack in its own building); a **hybrid** cloud is on-prem or private **plus** public, connected (customer data on-prem, the website on AWS); **multi-cloud** = **several** public providers at once (one Terraform can drive them all).

### Regions and Availability Zones
- A **region** = a **separate geographic area** where a provider has data centres, e.g. `us-east-1` (North Virginia) or `ap-south-1` (Mumbai). Each region is independent.
- An **Availability Zone (AZ)** = **one or more data centres inside a region**, with their **own power, cooling and network**, linked to the region's other AZs by very fast connections. Names add a letter: `ap-south-1a`, `ap-south-1b`.
- **High availability (HA)** = the app **keeps working when a part fails**. Put one web server in AZ `a` and another in AZ `b`, behind a load balancer: a fire or power cut in AZ `a` leaves AZ `b` serving users.
- Choose a region by **latency** (the delay to your users: closer is faster), laws about where data must be kept, price, and which services exist there.

### Networking in the cloud (the heart of this lecture)
- **VPC** (**Virtual Private Cloud**) = **your own isolated, private network** inside the provider's data centres; nothing gets in or out unless you allow it. Azure calls it a **VNet** (Virtual Network). It has a **CIDR block** (its address range), e.g. `10.0.0.0/16` = 65,536 private addresses, and it lives in **one region** but spans **all its AZs**.
- **Subnet** = a **smaller range carved out of the VPC's range**, living in **exactly one AZ**, e.g. `10.0.1.0/24` (256 addresses; AWS reserves 5, so 251 usable). A **public subnet** can be reached from the internet (web servers, load balancers); a **private subnet** can't (databases).
- **Internet gateway (IGW)** = the VPC's **front door to the internet**. One per VPC. On its own it does nothing: a route has to point to it.
- **Route table** = the signposts: a list of rules (**routes**), each "destination CIDR → target", saying where traffic goes next. `10.0.0.0/16 → local` (added automatically) keeps traffic for the VPC inside it; `0.0.0.0/0 → igw-…` sends **everything else** (`0.0.0.0/0` = any address) out through the internet gateway. When several routes match, the **most specific** one wins (`/16` beats `/0`).
- **NAT gateway** (Network Address Translation: many private addresses share one public one) = **out, but not in**. It sits in a **public** subnet, and a private subnet's route table sends `0.0.0.0/0` to it, so private servers can **start** connections out (to download updates), but nobody on the internet can start one in.

⚠️ **Trap:** a subnet is public **because its route table sends `0.0.0.0/0` to an internet gateway**, not because of its CIDR range. Public and private subnets both use 10.x addresses.
```
 ┌ VPC 10.0.0.0/16 ──────────── Internet Gateway ⇄ INTERNET ──────────────────────────────────┐
 │ Public subnet 10.0.1.0/24 (AZ a)              │ Private subnet 10.0.2.0/24 (AZ b)           │
 │ routes: 10.0.0.0/16 → local, 0.0.0.0/0 → IGW  │ routes: 10.0.0.0/16 → local, 0.0.0.0/0 → NAT │
 │ [EC2 web] ◄ SG: 22, 80    [NAT gateway] ──────┼──► [database] ◄ SG: 5432 from web SG only    │
 └────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Two firewalls: security groups and NACLs
A **firewall** filters traffic by rules, and AWS gives you two layers. A **security group (SG)** sits **on an instance** (one server); a **NACL** (**Network Access Control List**) sits **on a whole subnet**. **Stateful** means the firewall **remembers connections**: if a request was allowed in, its reply is allowed out automatically. **Stateless** means it remembers nothing, so replies need their own rule, usually for the **ephemeral ports** 1024–65535 (the temporary ports a client picks for its side of a connection).

| | **Security group** | **NACL** |
|---|---|---|
| Protects | **An instance** (your main firewall) | **A whole subnet** (a coarse extra layer) |
| Rules | **Allow only**; a new SG allows no inbound, all outbound. A rule can allow "from another SG" | **Allow and deny**, numbered (the default NACL allows everything) |
| Memory | **Stateful**: replies allowed automatically | **Stateless**: replies need their own rule |
| Evaluation | All rules considered together | Lowest number first; the first match wins |

**Analogy:** a security group is a **doorman at each flat's door** who remembers who went out and lets them back in. A NACL is the **checkpoint at the building's gate** with a printed list, checking everyone in both directions and remembering nobody.

### Compute, storage and identity
- **EC2** (**Elastic Compute Cloud**) = AWS's **virtual machines**; one VM is an **instance**, and its **instance type** is its size (e.g. `t3.micro`). An **AMI** (**Amazon Machine Image**) = the **template disk** it boots from (OS plus software), like a Docker image for a whole VM. A **key pair** = SSH keys (AWS puts the public key on the instance; you keep the private key to log in). **User data** = a script that runs **once, at first boot**.
- **S3** (**Simple Storage Service**) = **object storage**: a **bucket** (its name must be unique across all of AWS) holds **objects** (files), read and written over HTTP rather than attached as a disk. Used for backups, static websites and **Terraform state**.
- **IAM** (**Identity and Access Management**) = **who can do what** in your AWS account:
  - a **user** = a person or program with **long-term** credentials (a password or access keys); a **group** = users who share permissions
  - a **role** = permissions that a service, instance or pipeline **assumes temporarily**, with no stored keys, so roles are preferred over access keys
  - a **policy** = a **JSON list of permissions**, e.g. `{"Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::counter-assets/*"}` ("may read files from the `counter-assets` bucket, nothing else"). An **ARN** (**Amazon Resource Name**) is any resource's unique ID.
  - Everything is **denied by default**, and an explicit Deny beats any Allow. **Least privilege** = give each user or role **only the permissions it needs**. Lock the **root user** (the account owner's original login) away with **MFA** (Multi-Factor Authentication: a password plus a code from your phone).

### 🛠 Practical B (~15 min): build the network with Terraform, against LocalStack
**LocalStack** imitates AWS's APIs on your Mac at `http://127.0.0.1:4566`. We point Terraform's **real** `aws` provider at it, with **fake credentials**, so nothing reaches Amazon or costs money. ⚠️ The `aws` provider is a big download (~770 MB), so the first `init` takes a few minutes. Run `mkdir -p ~/devops-practice/tf-cloud && cd ~/devops-practice/tf-cloud && code main.tf`, then paste and save:
```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" } # the real AWS provider
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
    ec2 = "http://127.0.0.1:4566" # EC2's API also handles VPCs, subnets and SGs
    sts = "http://127.0.0.1:4566" # STS = Security Token Service, AWS's "who am I"
  }
}
resource "aws_s3_bucket" "files" { bucket = "practice-bucket" } # bucket names are unique across all of AWS
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16" # 65,536 private addresses
  tags       = { Name = "practice-vpc" }
}
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id # reference → built after the VPC
  cidr_block              = "10.0.1.0/24"   # a /24 slice of the VPC...
  availability_zone       = "us-east-1a"    # ...in exactly ONE AZ
  map_public_ip_on_launch = true            # instances here get a public IP
}
resource "aws_internet_gateway" "igw" { vpc_id = aws_vpc.main.id } # the VPC's door to the internet
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
  ingress { # inbound: HTTP from anyone
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress { # outbound: everything (Terraform removes AWS's default rule unless you write one)
    from_port   = 0
    to_port     = 0
    protocol    = "-1" # -1 = all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }
}
data "aws_availability_zones" "all" { state = "available" } # a DATA SOURCE: reads, creates nothing
output "azs" { value = data.aws_availability_zones.all.names }
```
```bash
terraform init && terraform validate && terraform apply   # Installed hashicorp/aws v6.x … Success! … Plan: 7 to add → type yes
curl -s http://127.0.0.1:4566/practice-bucket             # XML with <Name>practice-bucket</Name>: the bucket exists
```
- **The graph in action:** the VPC and the bucket are created **together** (neither references anything); everything else waits for the VPC. The output `azs` lists `us-east-1a` … `us-east-1f`, read by the data source.
- **An in-place update:** add `Env = "dev"` to the VPC's `tags`, and `terraform plan` shows `~ update in-place`, `0 to add, 1 to change`: a tag can change on a live VPC, so nothing is replaced.
- **Clean up:** `terraform destroy` (type yes; the VPC goes **last**), then `rm -rf .terraform` to free the ~770 MB.

**The server itself, on paper only** (LocalStack's free edition can't boot a real VM). In real AWS you'd add:
```hcl
resource "aws_instance" "web" {
  ami                    = "ami-0abc123example"        # the template disk (an Ubuntu image's ID)
  instance_type          = "t3.micro"                  # the size: 2 virtual CPUs, 1 GB RAM
  subnet_id              = aws_subnet.public.id        # inside the public subnet
  vpc_security_group_ids = [aws_security_group.web.id] # attach the web firewall
  key_name               = "vansh-laptop"              # an existing key pair, for SSH
  user_data              = "#!/bin/bash\napt-get update -y && apt-get install -y nginx"
}
```
*`user_data` is the script that runs once, at first boot: here it installs nginx.*
- **After apply,** an output of `aws_instance.web.public_ip` (known after apply) prints something like `54.x.x.x`.
- **How a request reaches nginx (draw this in exams):** browser → public IP → **internet gateway** → route table (`0.0.0.0/0 → IGW`) → public subnet → **security group** (port 80 allowed?) → EC2: nginx. The stateful SG lets the reply back out.
- **If it times out, check in this order:** does the instance have a public IP? Does the route table send `0.0.0.0/0` to the IGW? Does the SG allow the port? Is the app listening on `0.0.0.0` (not `127.0.0.1`)?

**Managed Kubernetes (awareness):** **EKS** (Amazon Elastic Kubernetes Service), **AKS** (Azure Kubernetes Service), and **GKE** (Google Kubernetes Engine) run the control plane for you; you pay for worker nodes in your subnets, and `type: LoadBalancer` gets a real cloud load balancer. Typical: Terraform builds the VPC + EKS, then Helm or Argo CD deploys the apps. **Same idea, other clouds:** Azure = Virtual Machines, Blob Storage, **VNet**, NSG (Network Security Group), AKS, ARM (Azure Resource Manager) templates / Bicep; Google = Compute Engine, Cloud Storage, VPC, firewall rules, GKE.
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
| **Workspaces** | Several **separate state files from the same code** (e.g. one for dev, one for prod): `terraform workspace new dev`, `workspace select prod`, `workspace list`. Every folder starts in `default` |
| **Modules** | A **folder of `.tf` files reused like a function**; your own folder is the **root module**. Use one with `module "vpc" { source = "terraform-aws-modules/vpc/aws" … }`, run `init`, then read its outputs as `module.vpc.vpc_id` |
| **`sensitive = true`** | Shows `(sensitive value)` on screen instead of the value, but ⚠️ it's **still plain text in the state**. An output that uses a sensitive variable must also say `sensitive = true` |
| **`lifecycle` block** | Inside a resource. **`prevent_destroy = true`**: any plan that would delete it fails (for databases). **`create_before_destroy = true`**: on replacement, build the new one first (`+/-`). **`ignore_changes = [tags]`**: ignore outside changes to those fields |
| **Version constraints** | `"~> 6.0"` = **≥ 6.0 and < 7.0** (any 6.x); `"~> 6.4.0"` = ≥ 6.4.0 and < 6.5.0 (patches only); `">= 1.5"` = that or newer; `"= 2.1.0"` = exactly. `~>` is called the "pessimistic constraint" |
| **`.gitignore`** · credentials | **Ignore** `.terraform/`, `*.tfstate`, `*.tfstate.*` (backups), `crash.log` and any `*.tfvars` holding secrets; **commit** your `.tf` files and **`.terraform.lock.hcl`**. Cloud keys never go in `.tf` files: use environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`), a profile, or, in GitHub Actions, an **OIDC** (OpenID Connect) role that gives the pipeline short-lived credentials |
| **OpenTofu** | In 2023 Terraform's licence became more restrictive (**BSL**, Business Source License, instead of open source), so the community **forked** it (copied the code to continue separately) as OpenTofu: `tofu init/plan/apply` |
| **Infracost / tfsec / Checkov** | Pipeline tools for Terraform code: Infracost comments a change's **monthly cost** on the PR; tfsec (now part of **Trivy**: `trivy config .`) and Checkov **scan IaC** for risks like public buckets or port 22 open to the world |
| **Shared responsibility** | The provider secures **"OF the cloud"** (buildings, hardware, hypervisors); **you** secure **"IN the cloud"** (your data, IAM, SG rules, encryption, and OS patches on IaaS) |
| **CIDR quick maths** | `/16` = 65,536 addresses · `/24` = 256 (**251** usable in AWS) · `/32` = exactly **one** address · `0.0.0.0/0` = **every** address |

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

### ✍️ Write the command
**C1.** In a brand-new Terraform folder: download the providers, rewrite the `.tf` files in the standard layout, then check the configuration's syntax and references. Each step should run only if the one before it worked.

**C2.** Make a plan with the variable `env` set to `prod` **on the command line**, and save it to a file named `tfplan` so that exactly this reviewed plan can be applied later.

**C3.** In a CI pipeline (nobody is there to type `yes`), apply the configuration in Practical A's folder; then, only if that worked, print **just the raw value** of the output `pet_name` (no quotes, no `pet_name =`).

**C4.** Before cleaning up, list the address of every resource Terraform is managing; then, only if that worked, delete all of them **without** the `yes` prompt.
