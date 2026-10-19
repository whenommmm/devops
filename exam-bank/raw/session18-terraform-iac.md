# Labs · Session 18 · Terraform and Infrastructure as Code (IaC)

IaC = Infrastructure as Code (servers, networks and files described in code files instead of clicked by hand). Terraform files use HCL (HashiCorp Configuration Language) and end in `.tf`. These labs use the `local` provider (writes files on your machine) and the `random` provider (generates random values), so they work anywhere with no cloud account.

## Lab 1 · Declare a provider, create a small resource, run fmt / init / validate / plan, explain the output
**Answer (what to write in the exam):**
```bash
mkdir tf-lab && cd tf-lab       # create main.tf (below) in this folder
terraform fmt                   # tidy formatting
terraform init                  # download the local + random providers
terraform validate              # check syntax and references
terraform plan                  # preview only: nothing is created yet
```
`main.tf`
```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "local" {}
provider "random" {}

resource "random_pet" "name" {
  length = 2
}

resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hello from ${random_pet.name.id}\n"
}
```
**What each line does:**
- `required_providers`: which plugins (providers) this code needs, where to download them from, and which versions are allowed (`~> 2.5` = 2.5 or newer, but below 3.0).
- `provider "local" {}` / `provider "random" {}`: turn the providers on (these two need no settings).
- `resource "random_pet" "name"`: creates a random name such as `happy-tiger`; the full address is `random_pet.name`.
- `resource "local_file" "hello"`: writes `hello.txt`. Using `${random_pet.name.id}` makes it depend on the pet, so Terraform creates the pet first.
- `terraform fmt`: rewrites `.tf` files in standard style (prints the names of the files it changed).
- `terraform init`: downloads the providers into `.terraform/` and writes `.terraform.lock.hcl`. Run it first, and again whenever you add a provider.
- `terraform validate`: checks syntax, types and references without contacting anything.
- `terraform plan`: compares your code with the state file (empty at first) and prints what it *would* do.

**Verify / read the output:**
- `validate` → `Success! The configuration is valid.`
- `plan` → `# local_file.hello will be created` and `# random_pet.name will be created`, each with a `+` in front.
  - `+ create` = new resource. Other symbols: `~` update in place, `-/+` destroy and re-create, `-` destroy.
  - `(known after apply)` = the value only exists after creation (for example, the random name).
  - Last line: `Plan: 2 to add, 0 to change, 0 to destroy.` = 2 new resources, nothing modified or deleted.
  - The note "You didn't use the -out option" means you can run `terraform plan -out=tfplan` and then `terraform apply tfplan` to apply exactly that plan.

**🧠 In simple words:** You describe what you want (a random name and a file) in `main.tf`. `init` downloads the plugins, `fmt` and `validate` check the code, and `plan` shows a dry-run list of changes without touching anything. In short: write `.tf`, then run fmt → init → validate → plan, and read the `+` lines and the "Plan: X to add" summary.

## Lab 2 · Use variables and outputs, pass values through a .tfvars file, print an output after apply
**Answer (what to write in the exam):**
```bash
terraform fmt && terraform init && terraform validate
terraform plan  -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"    # type: yes
terraform output                                 # list all outputs
terraform output -raw file_path                  # just the value, good for scripts
cat shop-test.txt
```
`variables.tf`
```hcl
variable "app_name" {
  description = "Name of the app"
  type        = string
}

variable "environment" {
  description = "dev, test or prod"
  type        = string
  default     = "dev"
}
```
`main.tf`
```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

resource "local_file" "config" {
  filename = "${path.module}/${var.app_name}-${var.environment}.txt"
  content  = "app=${var.app_name}\nenv=${var.environment}\n"
}
```
`outputs.tf`
```hcl
output "file_path" {
  description = "Where the config file was written"
  value       = local_file.config.filename
}
```
`terraform.tfvars`
```hcl
app_name    = "shop"
environment = "test"
```
**What each line does:**
- `variable "app_name"`: declares an input. It has no `default`, so it must be given a value (or Terraform asks for it).
- `var.app_name`: how the code reads a variable's value.
- `terraform.tfvars`: the values file. Terraform loads a file with exactly this name automatically. You need `-var-file=` for any other name, such as `-var-file=prod.tfvars`.
- Precedence (later wins): `default` → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` / `-var` on the command line.
- `output "file_path"`: prints a useful value after apply. Other tools can read it with `terraform output`.

**Verify:** `apply` ends with `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` and then `Outputs: file_path = "./shop-test.txt"`. `cat shop-test.txt` → `app=shop` / `env=test`.

**🧠 In simple words:** Variables make one piece of code reusable: the same `main.tf` can build dev, test or prod just by changing the `.tfvars` file. Outputs print the important results (like a file path or a server IP) after apply. In short: declare inputs in variables.tf, set them in a .tfvars file, and read results with `terraform output`.

## Lab 3 · Modify a resource, inspect the plan before applying, explain state, and keep secrets and state out of Git
**Answer (what to write in the exam):**
```bash
terraform apply                    # first build: random_pet.name + local_file.hello
# edit main.tf: change the file content (below)
terraform plan                     # ALWAYS read the plan before applying
terraform apply                    # type yes only if the plan is what you expect
terraform state list               # resources Terraform is tracking
terraform state show local_file.hello
git status                         # state and .terraform/ must NOT be listed
```
Change in `main.tf`:
```hcl
resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hi again from ${random_pet.name.id}\n"   # was "Hello from ..."
}
```
`.gitignore`
```gitignore
.terraform/
*.tfstate*
*.tfvars
tfplan
crash.log
```
**What the plan shows:**
- `-/+ resource "local_file" "hello"` with `~ content = "Hello…" -> "Hi again…" # forces replacement` → `Plan: 1 to add, 0 to change, 1 to destroy.` The provider cannot edit this attribute in place, so Terraform deletes the file and creates it again.
- `~ update in-place`: shown when an attribute can be changed without re-creating (for example, changing `tags` on a cloud virtual machine) → `Plan: 0 to add, 1 to change, 0 to destroy.`
- `random_pet.name` is not listed, because its config did not change.

**How state maps config to real infrastructure:**
- `terraform.tfstate` is a JSON (JavaScript Object Notation) file. For each resource address in the code (`local_file.hello`), it stores the real object's ID and attributes.
- `plan` = read the real objects (refresh) + compare them with your code + compare them with the state, then print the difference. Without state, Terraform would not know that it already created something.
- Don't commit state: it can hold secrets in plain text (passwords, keys), and two people with different local copies would overwrite each other. Teams keep state in a remote backend with locking (for example, an S3 bucket with locking, or Terraform Cloud).
- `*.tfvars` is ignored only when it holds secrets. Commit `.terraform.lock.hcl`, because it pins provider versions.

**Verify:** `terraform state list` → `local_file.hello` and `random_pet.name`. `git check-ignore -v terraform.tfstate` → prints the `.gitignore` rule that hides it.

**🧠 In simple words:** When you edit code, Terraform compares it with its state file and the real world, then shows a plan: `~` means "edit it", `-/+` means "delete and rebuild it". Reading the plan first stops surprise deletions. In short: edit → plan → read the symbols → apply, and keep `*.tfstate*`, `.terraform/` and secret `.tfvars` files out of Git.
