# Session 15 · Infrastructure as Code, Terraform & Cloud · FAST (12-min read)
*Two lectures in one: how Terraform builds infrastructure from text files, and the cloud networks and firewalls it usually builds.*

## 1 · Infrastructure as Code: declarative and idempotent
**Infrastructure** means the servers, networks and firewalls an app runs on. Building it by clicking around a cloud website cannot be repeated exactly and leaves no record of changes. **IaC (Infrastructure as Code)** fixes this: you describe the infrastructure in text files kept in Git, and a tool makes the real infrastructure match them.
- **Imperative** code lists the steps ("create a server, then create another"), and the tool runs exactly those steps.
- **Declarative** code describes only the end result ("2 servers should exist"); the tool compares that with what exists and works out the steps itself. Terraform is declarative.
- An action is **idempotent** when doing it many times gives the same result as doing it once, like a lift button pressed five times still bringing one lift.
- Declarative tools are idempotent: if the file says 2 servers and 2 exist, the next run changes nothing and Terraform prints `No changes.`, whereas an imperative script that creates 2 servers leaves 4 after two runs.

⚠️ **Exam trap:** plain `mkdir data` is not idempotent (its second run fails with "File exists"), whereas `mkdir -p data` is.

## 2 · Terraform's architecture: core, providers and HCL
**Terraform** is a command-line tool that builds infrastructure on almost any cloud from text files. **Terraform core** reads your files, works out the order, compares them with what exists and makes a plan, but cannot talk to any cloud itself. **Providers** are plugins that each know one platform's **API** (Application Programming Interface: the way programs ask a service to do things), such as `aws` (Amazon Web Services) or `azurerm` (Microsoft Azure); core decides what to create, and the provider makes the calls, like a travel agent booking through each airline.
- Files are written in **HCL** (HashiCorp Configuration Language) and end in `.tf`.
- A **variable** is an input, read as `var.NAME`, that takes the strongest value given; from weakest to strongest: its `default`, an environment variable `TF_VAR_name`, a `terraform.tfvars` file (loaded automatically), and `-var` or `-var-file` on the command line.

```hcl
variable "env" {
  default = "dev"
}
resource "aws_s3_bucket" "logs" {
  bucket = "${var.env}-logs"
}
output "bucket_name" {
  value = aws_s3_bucket.logs.bucket
}
```
- A **resource** is something Terraform creates and manages: `aws_s3_bucket` is its type, `logs` its local name (a nickname inside the code), and `${ }` puts a value into a string, giving `dev-logs`.
- An **output** is printed after `apply`: `bucket_name = "dev-logs"`.
- One resource refers to another as `TYPE.LOCAL_NAME.ATTRIBUTE`; Terraform builds the referenced resource first and unrelated ones in parallel.

⚠️ **Exam trap:** `TF_VAR_env` beats only the default, whereas `terraform.tfvars` beats `TF_VAR_env`, and `-var` beats everything.

## 3 · The workflow (init, plan, apply, destroy) and the state file
Terraform's workflow is four commands, run in the folder holding your `.tf` files. `plan` is like a builder's written quote (nothing is touched yet), and `apply` is signing it.
- `terraform init` downloads the providers into a hidden `.terraform/` folder, sets up where the state is kept, and writes the lock file `.terraform.lock.hcl`; without it, `plan` fails and asks you to run `init`.
- `terraform plan` is a dry run: it **refreshes** (reads the real settings of every resource listed in the state), compares them with the code, and prints what `apply` would do.
- `terraform apply` shows a fresh plan and continues only if you type `yes`; `-auto-approve` skips it in pipelines.
- `terraform destroy` deletes everything recorded in the state, in reverse order, and never touches anything created by hand.
- In a plan, `+` is create, `-` is destroy, `~` is update in place (a tag change), and `-/+` is replace: delete the old object, then create a new one (`# forces replacement`). A replacement counts as 1 to add plus 1 to destroy.

After the first apply, Terraform writes **`terraform.tfstate`**, a file recording every resource it manages and the real cloud ID each one maps to. It is Terraform's only memory of what it owns: if the state is lost, Terraform forgets those resources, and the next plan wants to create them all again as duplicates.
- Never edit the state by hand, and never commit it to Git: it stores secrets such as passwords in plain text, and copies on different laptops go out of date.
- Teams keep the state in a **remote backend**, shared storage such as an Amazon S3 (Simple Storage Service) bucket, so everyone uses one copy.
- With **state locking**, a second run started during an apply stops with `Error acquiring the state lock` instead of corrupting it.
- **Drift** is a manual change to real infrastructure; the next plan notices it and plans to put things back as the code says.

⚠️ **Exam trap:** `-/+` deletes the old object along with its data and address, whereas `~` modifies the same object. Terraform never rewrites your `.tf` files to match a manual change; it changes reality to match the code.

## 4 · Cloud fundamentals: service models, regions and zones
**Cloud computing** means renting someone else's computers, storage and networks over the internet, on demand, instead of buying your own **on-premises** servers (in your own building) months ahead. Like electricity, you pay per unit used (**pay-as-you-go**), and **elasticity** adds servers automatically when demand rises and removes them when it falls.
- In **IaaS (Infrastructure as a Service)** the provider runs the hardware and network, while you manage the OS (operating system), runtime, app and data; examples are AWS EC2 (Elastic Compute Cloud) virtual machines.
- In **PaaS (Platform as a Service)** you manage only your app and data, while the provider runs the OS, runtime and scaling; examples are Heroku and AWS Elastic Beanstalk.
- In **SaaS (Software as a Service)** you simply use finished software; examples are Gmail and Microsoft 365.
- A **region** is a separate geographic area of data centres, such as `ap-south-1` (Mumbai), whereas an **AZ (Availability Zone)** is one or more data centres inside a region with their own power and network, such as `ap-south-1a`.
- **High availability** means the app keeps working when a part fails: servers in two AZs behind a load balancer survive the loss of one AZ.

⚠️ **Exam trap:** with IaaS you patch the operating system yourself, whereas with PaaS the provider does it. Two servers in the same AZ are not highly available, even though their region has other AZs.

## 5 · Cloud networking, built with Terraform
A **VPC (Virtual Private Cloud)** is your own isolated network inside the cloud (Azure calls it a **VNet**, Virtual Network). Its address range, such as `10.0.0.0/16`, spans all the AZs of one region.
- A **subnet** is a smaller slice of the VPC's range, such as `10.0.1.0/24`, in exactly one AZ. A **public subnet** can be reached from the internet (web servers), whereas a **private subnet** cannot (databases).
- An **internet gateway** is the VPC's door to the internet, but it does nothing until a route points to it.
- A **route table** lists routes of the form "destination → target": `10.0.0.0/16 → local` keeps traffic inside the VPC, and `0.0.0.0/0 → internet gateway` sends everything else to the internet. A route table affects a subnet only once it is associated (attached) with it; until then the subnet uses the VPC's main table, which has only the local route.
- A **NAT (Network Address Translation) gateway** sits in a public subnet and lets private-subnet servers start connections out, while nobody outside can start one in.
- A **security group** is a firewall around each server with **allow rules only**. It is **stateful**: it remembers allowed connections, so replies to allowed requests go out without any outbound rule.

To **provision** (create) this network with Terraform, each piece is a resource, and references set the build order:
```hcl
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```
- `vpc_id = aws_vpc.main.id` puts the subnet inside the VPC, so Terraform creates the VPC first.
- The association attaches the route table holding `0.0.0.0/0 → internet gateway` to the subnet; this is what makes the subnet public.

⚠️ **Exam trap:** a subnet is public because its associated route table sends `0.0.0.0/0` to an internet gateway, not because of its address range. A security group (per server, stateful, allow only) differs from a NACL (Network Access Control List: per subnet, stateless, allow and deny).

## ⚡ Must-remember
| Term / command | Meaning |
|---|---|
| `terraform init` | Downloads providers and writes the lock file |
| `terraform plan` | Dry run: shows what would change, changes nothing |
| `apply` / `destroy` | Build after `yes` / delete everything in the state, in reverse |
| `~` vs `-/+` | Update in place vs delete then recreate (1 add + 1 destroy) |
| `terraform.tfstate` | Never edit or commit it (secrets in plain text) |
| Variable sources | `default` < `TF_VAR_` < `terraform.tfvars` < `-var` |
| IaaS / PaaS / SaaS | EC2 (you patch the OS) / Heroku (you bring code) / Gmail |
| Public subnet | Route table sends `0.0.0.0/0` to an internet gateway |
| Security group | Per-server firewall: allow rules only, stateful |

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")

**1.** Riya's old shell script said "create a server" twice, so every run of it added 2 more servers. She replaces it with a Terraform file that declares exactly 2 servers, runs `terraform apply` on Monday (2 servers are created), and runs it again on Tuesday with the file unchanged and nobody having touched the servers. What happens on Tuesday?
- A) It creates 2 more servers, since apply runs the file's steps again
- B) It replaces both servers (-/+), so they freshly match the file
- C) It reports no changes, since the 2 servers already match the file
- D) It stops with an error, since servers with those names already exist

**2.** A folder contains this code and also a `terraform.tfvars` file with the line `stage = "beta"`:
```hcl
variable "stage" {
  default = "dev"
}
resource "aws_s3_bucket" "site" {
  bucket = "shop-${var.stage}"
}
output "name" {
  value = aws_s3_bucket.site.bucket
}
```
You run `export TF_VAR_stage=staging`, then `terraform apply -var 'stage=prod'`, and type `yes`. What does the output `name` show now, and what would it have shown if you had run the same apply **without** the `-var` flag? (now / without `-var`)
- A) `shop-prod` / `shop-staging`
- B) `shop-prod` / `shop-beta`
- C) `shop-staging` / `shop-staging`
- D) `shop-beta` / `shop-beta`

**3.** Arjun's Terraform folder keeps its state only on his laptop (no remote backend) and manages a live VPC. Tidying up, he deletes `terraform.tfstate`, thinking it is a log file; the `.tf` files are untouched and the VPC is still running. He then runs `terraform plan`. What does it show?
- A) No changes, since plan refreshes by reading the real cloud
- B) A plan to destroy the VPC, since it is missing from the state
- C) An error, since the state lock was deleted along with the file
- D) A plan to create a VPC, since Terraform forgot it owns one

**4.** A shop runs its website on two AWS EC2 virtual machines behind a load balancer, both placed in `ap-south-1a`. Its engineers install operating-system security patches on the machines every month. A power cut takes out the `ap-south-1a` data centres, while `ap-south-1b` keeps running. Which statement is right about the service model and the outcome?
- A) IaaS; the site goes down, as both servers sit in one AZ
- B) PaaS; the site goes down, as both servers sit in one AZ
- C) IaaS; the site stays up, as the region still has AZ 1b running
- D) PaaS; the site stays up, as the provider runs servers for you

**5.** A Terraform folder contains exactly these resources, all applied without errors: a VPC; one subnet `10.0.1.0/24`; an internet gateway; a route table with the route `0.0.0.0/0 → internet gateway`; a security group allowing inbound port 80 from `0.0.0.0/0` (no outbound rules); and an EC2 instance in the subnet with a public address, running a web server on port 80. Browser requests to the instance time out. What is missing?
- A) An outbound security group rule, so that replies can get out
- B) A NAT gateway in the subnet, so that requests can come in
- C) An association that attaches the route table to the subnet
- D) A subnet range outside 10.x, so the subnet counts as public
