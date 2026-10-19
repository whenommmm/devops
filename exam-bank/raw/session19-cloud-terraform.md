# Labs · Session 19 · Cloud networking with Terraform

Acronyms: VPC = Virtual Private Cloud (your own private network inside AWS, Amazon Web Services). CIDR = Classless Inter-Domain Routing (an IP range written like `10.0.0.0/16`). IGW = Internet Gateway. NAT = Network Address Translation. SG = Security Group (a firewall around a server). EC2 = Elastic Compute Cloud (AWS virtual machines). AMI = Amazon Machine Image (the operating-system image a VM boots from). AZ = Availability Zone (one data-centre location inside a region). Azure uses the names VNet and NSG (Network Security Group) for the same ideas.

## Lab 1 · Design a VPC with public and private subnets, route tables and security rules; draw it and explain the traffic flow
**Answer (model diagram):**
```
                     Internet (users, admin laptop)
                                 |
                     [ Internet Gateway (IGW) ]
                                 |
+---------------------- VPC 10.0.0.0/16 --------------------------+
|                                                                 |
|  PUBLIC subnet 10.0.1.0/24       PRIVATE subnet 10.0.2.0/24     |
|  +-------------------------+     +-------------------------+    |
|  | web server   [SG-web]   |---->| app / DB    [SG-app]    |    |
|  | NAT gateway (Elastic IP)|<----| (no public IP)          |    |
|  +-------------------------+     +-------------------------+    |
|  Route table "public"            Route table "private"          |
|   10.0.0.0/16 -> local            10.0.0.0/16 -> local          |
|   0.0.0.0/0   -> IGW              0.0.0.0/0   -> NAT gateway    |
+-----------------------------------------------------------------+
```
**Security rules:**
| Security group | Inbound allowed | Outbound |
|---|---|---|
| SG-web (public subnet) | TCP 80 and 443 from `0.0.0.0/0`. TCP 22 (SSH, Secure Shell) **only** from the admin's IP, e.g. `203.0.113.10/32` | all |
| SG-app (private subnet) | app port 8080 **only** from SG-web (a security group is the source, not an IP range) | all (leaves through the NAT gateway) |

**Traffic flow:**
1. **User → web:** internet → IGW → public route table → web server. SG-web allows port 443.
2. **Web → app:** web server → app on 8080 through the `local` route (traffic stays inside the VPC). SG-app allows it only because the source is SG-web.
3. **Private server → internet (updates):** app → private route table `0.0.0.0/0` → NAT gateway in the public subnet → IGW → internet. Replies come back the same way.
4. **Blocked:** the internet cannot start a connection to the private subnet (the servers have no public IP, and NAT only lets out connections that started inside). SSH from any IP except the admin `/32` is dropped by SG-web.
- Security groups are *stateful*: reply traffic is allowed automatically. NACLs (Network Access Control Lists) are optional, *stateless* filters at the subnet level.
- A subnet is "public" only because its route table sends `0.0.0.0/0` to an IGW.

**🧠 In simple words:** The VPC is your private network. The public subnet holds things the internet must reach, and the private subnet hides the app and database. Route tables decide where traffic goes (IGW for public, NAT for private outbound only), and security groups decide which ports are allowed. In short: public = route to IGW, private = route to NAT, and SGs open only the ports you need.

## Lab 2 · Use Terraform to build a small VPC and one cheap VM in a sandbox account; plan before apply, then destroy
**Answer (what to write in the exam):**
```bash
aws configure --profile sandbox        # keys are saved in ~/.aws, never in .tf files
export AWS_PROFILE=sandbox
terraform fmt && terraform init && terraform validate
terraform plan -var="admin_cidr=203.0.113.10/32" -out=tfplan
terraform apply tfplan                 # applies exactly the reviewed plan
terraform output public_ip
curl http://$(terraform output -raw public_ip)
terraform destroy -var="admin_cidr=203.0.113.10/32"   # type yes
```
`main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region # credentials come from `aws configure` / env vars, never from this file
}

variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "admin_cidr" {
  type = string # e.g. "203.0.113.10/32"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "lab-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  tags                    = { Name = "lab-public" }
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  tags       = { Name = "lab-private" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name   = "lab-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  tags                   = { Name = "lab-web" }
  user_data              = <<-EOT
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
  EOT
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```
**What each line does:**
- `provider "aws"`: chooses the region. Credentials come from `aws configure` or environment variables, so they are never written in code.
- `aws_vpc` / `aws_subnet`: the network (`/16` = 65,536 addresses) and two `/24` slices (256 addresses each). `map_public_ip_on_launch` gives VMs in the public subnet a public IP.
- `aws_internet_gateway` + `aws_route_table` (`0.0.0.0/0 → igw`) + `aws_route_table_association`: these three make the subnet public. The private subnet has no such route. (A NAT gateway is skipped in a sandbox because it is billed per hour.)
- `aws_security_group`: allows 80 and 443 from anywhere, and SSH only from `var.admin_cidr`. `egress` allows all outbound traffic (Terraform removes AWS's default egress rule, so you must add it).
- `data "aws_ami"`: looks up the newest Amazon Linux 2023 image. `t3.micro` is a free-tier-sized VM. `user_data` installs nginx (a web server) at first boot.
- `plan -out=tfplan` then `apply tfplan`: review first, then apply exactly what you reviewed. `destroy` deletes everything in the state, so nothing keeps costing money.

**Verify:** the plan ends with `Plan: 8 to add, 0 to change, 0 to destroy.`, apply with `Apply complete! Resources: 8 added`, `curl` returns the nginx welcome page, and destroy ends with `Destroy complete! Resources: 8 destroyed.`

**🧠 In simple words:** One `main.tf` describes the whole network and the VM, so anyone can rebuild the same thing with one command. Planning first shows exactly what will be created, and destroy cleans up so the sandbox bill stays at zero. In short: write HCL → init → plan → apply → test → destroy, with credentials kept outside the code.

## Lab 3 · Security group that allows only the app port; prove SSH is not open to the whole internet
**Answer (what to write in the exam):**
```bash
MYIP=$(curl -s https://checkip.amazonaws.com)          # your public IP
terraform plan  -var="admin_cidr=${MYIP}/32" -out=tfplan
terraform apply tfplan
aws ec2 describe-security-groups --filters Name=group-name,Values=lab-web-sg \
  --query 'SecurityGroups[0].IpPermissions[].{port:FromPort,from:IpRanges[0].CidrIp}' --output table
curl -I http://<public_ip>             # app port works from anywhere
nc -zv -w 5 <public_ip> 22             # from YOUR IP: succeeded
nc -zv -w 5 <public_ip> 22             # from another network (phone hotspot): timed out
terraform plan -var="admin_cidr=0.0.0.0/0"   # rejected by the validation rule
```
Security rules in `main.tf`:
```hcl
variable "admin_cidr" {
  description = "Your own public IP as /32, e.g. 203.0.113.10/32"
  type        = string
  validation {
    condition     = endswith(var.admin_cidr, "/32")
    error_message = "admin_cidr must be ONE IP (/32), never 0.0.0.0/0."
  }
}

resource "aws_security_group" "web" {
  name   = "lab-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "SSH only from admin IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
**What each line does:**
- Ingress on port 80 from `0.0.0.0/0`: the only public port, because it is the application port. Add a 443 block the same way if the app uses HTTPS.
- Ingress on port 22 with `[var.admin_cidr]`: SSH is allowed from one IP only (`/32` = exactly one address). Every port not listed is denied by default.
- `validation { … }`: Terraform refuses any value that is not a single `/32`, so `0.0.0.0/0` (the whole internet) can never be applied by mistake.
- `describe-security-groups --query …`: prints the real rules that AWS stored, as evidence.
- `nc -zv -w 5 host 22`: tries a TCP connection with a 5-second timeout. "succeeded" means open; "timed out" means blocked.

**Verify:** the table shows `80 → 0.0.0.0/0` and `22 → <your IP>/32` (never `22 → 0.0.0.0/0`). `curl -I` → `HTTP/1.1 200 OK`. Port 22 from another network times out. The last plan fails with `Invalid value for variable … never 0.0.0.0/0`.

**🧠 In simple words:** A security group is an allow-list: anything not listed is blocked. Opening only port 80 to everyone and port 22 to your single IP means attackers scanning the internet cannot even try to log in over SSH. In short: app port open to `0.0.0.0/0`, SSH only from `your-ip/32`, proven with `describe-security-groups` and a blocked `nc` test.
