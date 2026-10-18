# Answer key · Session 15 FAST · Infrastructure as Code, Terraform & Cloud

**1.** Riya's old shell script said "create a server" twice, so every run of it added 2 more servers. She replaces it with a Terraform file that declares exactly 2 servers, runs `terraform apply` on Monday (2 servers are created), and runs it again on Tuesday with the file unchanged and nobody having touched the servers. What happens on Tuesday?
- A) It creates 2 more servers, since apply runs the file's steps again
- B) It replaces both servers (-/+), so they freshly match the file
- C) It reports no changes, since the 2 servers already match the file
- D) It stops with an error, since servers with those names already exist

**Correct: C.** Terraform is declarative: the file describes the end result ("2 servers exist"), and every run compares that with what really exists. On Tuesday both servers are already there and unchanged, so there is nothing to do and Terraform prints `No changes.`, which is exactly what idempotent means (many runs give the same result as one). A describes the old imperative script, which blindly repeats its steps; Terraform has no "steps" to repeat. B is wrong because a replacement (`-/+`) only happens when something changed in a way that forces it, and nothing changed. D is how a non-idempotent command such as plain `mkdir data` behaves on its second run; a declarative tool checks what exists first, so "already exists" simply means "nothing to do".

---

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

**Correct: B.** A variable takes the strongest value given, and from weakest to strongest the order is: `default` (dev), then the environment variable `TF_VAR_stage` (staging), then the `terraform.tfvars` file (beta), then `-var` on the command line (prod). With `-var 'stage=prod'`, the command line wins, `${var.stage}` puts `prod` into the string, and the output (which reads the bucket's `bucket` attribute) shows `shop-prod`. Without `-var`, the strongest remaining source is `terraform.tfvars`, so it shows `shop-beta`. A is the classic trap: it assumes the environment variable beats the file, but `TF_VAR_` beats only the default. C assumes the environment variable beats everything, even the command line. D assumes the `terraform.tfvars` file beats `-var`, but the command line is always the strongest.

---

**3.** Arjun's Terraform folder keeps its state only on his laptop (no remote backend) and manages a live VPC. Tidying up, he deletes `terraform.tfstate`, thinking it is a log file; the `.tf` files are untouched and the VPC is still running. He then runs `terraform plan`. What does it show?
- A) No changes, since plan refreshes by reading the real cloud
- B) A plan to destroy the VPC, since it is missing from the state
- C) An error, since the state lock was deleted along with the file
- D) A plan to create a VPC, since Terraform forgot it owns one

**Correct: D.** The state file is Terraform's only memory of which real resources it owns. With the state gone, Terraform sees a VPC in the code but no VPC in its records, so it plans `+ create`; applying that would build a second, duplicate VPC next to the running one. A is tempting but wrong: refreshing only re-reads the real settings of resources already listed in the state, and with an empty state there is nothing to re-read, so Terraform never goes looking for the old VPC. B gets it backwards: Terraform only deletes what is recorded in its state, and the VPC is now unknown to it (and still wanted by the code). C is wrong because a state lock exists only while a run is in progress; deleting the state file does not cause a lock error.

---

**4.** A shop runs its website on two AWS EC2 virtual machines behind a load balancer, both placed in `ap-south-1a`. Its engineers install operating-system security patches on the machines every month. A power cut takes out the `ap-south-1a` data centres, while `ap-south-1b` keeps running. Which statement is right about the service model and the outcome?
- A) IaaS; the site goes down, as both servers sit in one AZ
- B) PaaS; the site goes down, as both servers sit in one AZ
- C) IaaS; the site stays up, as the region still has AZ 1b running
- D) PaaS; the site stays up, as the provider runs servers for you

**Correct: A.** Renting EC2 virtual machines and patching their operating system yourself is IaaS (Infrastructure as a Service): the provider runs only the hardware and network. Both servers sit in the same Availability Zone, so when that AZ fails, both fail together and the load balancer has no healthy server to send users to; high availability needs servers spread across two AZs. B has the right outcome but the wrong model: with PaaS the provider patches the operating system, so the engineers would not be doing it. C has the right model but the wrong outcome: the region's other AZ only helps if a server is actually running there, and the servers do not move by themselves. D has both the model and the outcome wrong.

---

**5.** A Terraform folder contains exactly these resources, all applied without errors: a VPC; one subnet `10.0.1.0/24`; an internet gateway; a route table with the route `0.0.0.0/0 → internet gateway`; a security group allowing inbound port 80 from `0.0.0.0/0` (no outbound rules); and an EC2 instance in the subnet with a public address, running a web server on port 80. Browser requests to the instance time out. What is missing?
- A) An outbound security group rule, so that replies can get out
- B) A NAT gateway in the subnet, so that requests can come in
- C) An association that attaches the route table to the subnet
- D) A subnet range outside 10.x, so the subnet counts as public

**Correct: C.** A route table affects a subnet only once it is associated with it. Without an `aws_route_table_association` resource, the subnet keeps using the VPC's main route table, which has only the local route (`10.0.0.0/16 → local`), so there is no path between the subnet and the internet gateway and the subnet behaves as private. Adding the association attaches the table holding `0.0.0.0/0 → internet gateway` and makes the subnet public. A is wrong because a security group is stateful: replies to an allowed inbound request go back out automatically, with no outbound rule. B is wrong because a NAT gateway lets private servers start connections out but never lets the internet start connections in. D is wrong because a subnet is public because of its route table, not its address range; public subnets use 10.x addresses too.

---
