---
title: "Terraform on OCI"
seoTitle: "Terraform on OCI: Beginner's Guide + Real Examples"
seoDescription: "Step-by-step Terraform on OCI guide: provision VCN, subnets, and a compute instance from scratch. Includes full HCL code examples and best practices.	"
datePublished: 2026-03-03T15:57:47.937Z
cuid: cmmaskh7d00662fju2wa51ze2
slug: terraform-on-oci
cover: https://cdn.hashnode.com/uploads/covers/66070b822018a4a9c83bbb3a/39e8400b-f8c6-4a99-b462-649cee6fddb3.png
tags: devops, oci, terraform, infrastructure-as-code, oracle-cloud

---

## Introduction: Why Terraform on OCI?

If you've been clicking around the OCI Console to create VMs, networks, and databases — you're not alone. Most people start that way. But once you need to repeat that setup for a second environment, hand it off to a colleague, or recover from an accidental deletion, you quickly realize: clicking through a UI is not infrastructure management. It's infrastructure guessing.

That's where Terraform comes in. Terraform is an Infrastructure as Code (IaC) tool that lets you describe your cloud infrastructure in simple text files, then apply, change, or destroy it in a repeatable, auditable way. On Oracle Cloud Infrastructure, Terraform is a first-class citizen — Oracle maintains the official OCI provider, and the latest version (6.x) supports virtually every OCI service.

This guide is for complete beginners. We'll start from zero — what Terraform is, how it thinks, how to connect it to OCI — and build up to a real working example: a fully functional VCN with subnets, an Internet Gateway, and a Compute instance you can SSH into. Every line of code is explained.

> **🎯  What You'll Build By the End**
> 
> A working OCI environment built entirely with Terraform: VCN + Public subnet + Internet Gateway + Route Table + Security list + Ubuntu compute instance with SSH access. Everything destroyed with one command when you're done.

## Section 1: What Is Terraform and How Does It Work?

Terraform, built by HashiCorp, is an open-source IaC tool that uses a declarative approach to infrastructure management. That means you tell Terraform what you want — not how to build it. You write configuration files in HashiCorp Configuration Language (HCL), and Terraform figures out the sequence of API calls needed to make reality match your description.

### The Terraform Workflow: Plan, Apply, Destroy

Terraform operates in a simple three-command loop that you'll use constantly:

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Command</strong></p></td><td colspan="1" rowspan="1"><p><strong>What It Does</strong></p></td><td colspan="1" rowspan="1"><p><strong>When to Use</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>terraform init</p></td><td colspan="1" rowspan="1"><p>Downloads the OCI provider plugin, sets up the working directory</p></td><td colspan="1" rowspan="1"><p>First time, or after changing providers</p></td></tr><tr><td colspan="1" rowspan="1"><p>terraform plan</p></td><td colspan="1" rowspan="1"><p>Shows exactly what will be created, changed, or deleted — no real changes made</p></td><td colspan="1" rowspan="1"><p>Before every apply, to review the diff</p></td></tr><tr><td colspan="1" rowspan="1"><p>terraform apply</p></td><td colspan="1" rowspan="1"><p>Executes the plan and creates/modifies/destroys real OCI resources</p></td><td colspan="1" rowspan="1"><p>After reviewing the plan</p></td></tr><tr><td colspan="1" rowspan="1"><p>terraform destroy</p></td><td colspan="1" rowspan="1"><p>Tears down all resources described in your configuration</p></td><td colspan="1" rowspan="1"><p>Cleanup / decommission</p></td></tr><tr><td colspan="1" rowspan="1"><p>terraform fmt</p></td><td colspan="1" rowspan="1"><p>Auto-formats your .tf files to standard style</p></td><td colspan="1" rowspan="1"><p>Before committing code</p></td></tr><tr><td colspan="1" rowspan="1"><p>terraform validate</p></td><td colspan="1" rowspan="1"><p>Checks configuration syntax without connecting to OCI</p></td><td colspan="1" rowspan="1"><p>After writing new config</p></td></tr></tbody></table>

### Terraform State: The Source of Truth

Terraform maintains a state file (terraform.tfstate) that maps your configuration to real OCI resources. This is how Terraform knows what already exists and what needs to change. By default it's a local file, but for team environments you should store it remotely — OCI Object Storage works perfectly for this.

> **⚠️  Never Manually Edit the State File**
> 
> The terraform.tfstate file is Terraform's internal database. Editing it manually corrupts it. If you need to modify state, use 'terraform state' subcommands. Also — add terraform.tfstate to your .gitignore, as it can contain sensitive values.

### Key Terraform Concepts at a Glance

•      **Provider** — A plugin that enables Terraform to talk to a specific platform (in our case, oracle/oci)

•      **Resource** — A cloud object to create (e.g. oci\_core\_vcn, oci\_core\_instance)

•      **Data Source** — Read-only query for existing OCI data (e.g. get all availability domains)

•      **Variable** — Parameterize your config so it's reusable across environments

•      **Output** — Print useful values after apply (e.g. the new instance's public IP)

•      **Module** — A reusable group of resources, like a blueprint for a network pattern

## Section 2: Installing Terraform and Connecting to OCI

### Step 1: Install Terraform

Terraform is a single binary — no dependencies. Install the latest version for your OS:

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>OS</strong></p></td><td colspan="1" rowspan="1"><p><strong>Install Method</strong></p></td><td colspan="1" rowspan="1"><p><strong>Command / Link</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>macOS</p></td><td colspan="1" rowspan="1"><p>Homebrew</p></td><td colspan="1" rowspan="1"><p>brew tap hashicorp/tap &amp;&amp; brew install hashicorp/tap/terraform</p></td></tr><tr><td colspan="1" rowspan="1"><p>Ubuntu/Debian</p></td><td colspan="1" rowspan="1"><p>APT</p></td><td colspan="1" rowspan="1"><p>sudo apt-get install terraform (after adding HashiCorp repo)</p></td></tr><tr><td colspan="1" rowspan="1"><p>Windows</p></td><td colspan="1" rowspan="1"><p>Chocolatey</p></td><td colspan="1" rowspan="1"><p>choco install terraform</p></td></tr><tr><td colspan="1" rowspan="1"><p>Any</p></td><td colspan="1" rowspan="1"><p>Direct download</p></td><td colspan="1" rowspan="1"><p>developer.hashicorp.com/terraform/downloads</p></td></tr></tbody></table>

Verify the installation:

`terraform -version # Expected output: Terraform v1.9.x (or higher)`

### Step 2: Set Up OCI API Key Authentication

Terraform authenticates to OCI using an API key pair. You'll create an RSA key, upload the public key to OCI, and configure a local credentials file. This is a one-time setup.

1.     Log into the OCI Console

2.     Click your profile icon (top-right) → My Profile

3.     Scroll to API Keys → Add API Key → Generate API Key Pair

4.     Download the Private Key (.pem file) → click Add

5.     Copy the configuration snippet shown — you'll need these values

Create the OCI credentials file:

`mkdir -p ~/.oci`

`# Move your downloaded private key:`

`mv ~/Downloads/your-key.pem ~/.oci/oci_api_key.pem`

`chmod 400 ~/.oci/oci_api_key.pem`

`# Create the config file:`

`vi ~/.oci/config`

Paste the following into ~/.oci/config (use the values from the OCI console snippet):

\[DEFAULT\]

`user=ocid1.user.oc1..aaaaaaaaxxx          # Your User OCID`

`tenancy=ocid1.tenancy.oc1..aaaaaaaaxxx    # Your Tenancy OCID`

`region=ap-mumbai-1                         # Your home region`

`fingerprint=aa:bb:cc:dd:...               # Key fingerprint from OCI console`

`key_file=~/.oci/oci_api_key.pem          # Path to your private key`

> **💡  Finding Your OCIDs**
> 
> Tenancy OCID: OCI Console → top-right profile → Tenancy. User OCID: Profile → My Profile. Compartment OCID: Identity & Security → Compartments → click your compartment. OCIDs start with 'ocid1.' and are long strings — copy them carefully.

### Step 3: Verify the OCI CLI Connection

Before writing any Terraform, confirm your credentials work:

`# Install OCI CLI if not already installed:`

`pip install oci-cli --break-system-packages`

`# Test authentication:`

`oci iam region list --output table`

`# Should list all OCI regions — if it does, credentials are working`

## Section 3: Your First Terraform Project — File Structure

Good Terraform projects split configuration across multiple files by concern. Here's the recommended structure for a beginner OCI project:

> oci-terraform-demo/
> 
> ├── [provider.tf](http://provider.tf)       # OCI provider configuration and authentication
> 
> ├── [variables.tf](http://variables.tf)      # Input variable declarations
> 
> ├── terraform.tfvars  # Variable values (DO NOT commit to Git)
> 
> ├── [main.tf](http://main.tf)           # Core resources: VCN, subnets, gateways
> 
> ├── [compute.tf](http://compute.tf)        # Compute instance resources
> 
> └── [outputs.tf](http://outputs.tf)       # Output values to display after apply

Create the project directory and open it:

<table style="min-width: 25px;"><colgroup><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><code>mkdir oci-terraform-demo &amp;&amp; cd oci-terraform-demo</code></p></td></tr></tbody></table>

[**provider.tf**](http://provider.tf) **— Connecting Terraform to OCI**

This file tells Terraform which provider to use and how to authenticate. Create [provider.tf](http://provider.tf) with the following content:

> `terraform {`
> 
>   `required_providers {`
> 
>     `oci = {`
> 
>       `source  = "oracle/oci"`
> 
>       `version = ">= 6.0.0"    # Always pin a minimum version`
> 
>     `}`
> 
>   `}`
> 
>   `required_version = ">= 1.9.0"`
> 
> `}`
> 
> `provider "oci" {`
> 
>   `tenancy_ocid     = var.tenancy_ocid`
> 
>   `user_ocid        = var.user_ocid`
> 
>   `fingerprint      = var.fingerprint`
> 
>   `private_key_path = var.private_key_path`
> 
>   `region           = var.region`
> 
> `}`

> **🔒  Never Hardcode Credentials**
> 
> Notice we use variables (var.tenancy\_ocid) instead of pasting OCIDs directly. The actual values go in terraform.tfvars which you add to .gitignore. This prevents accidental credential exposure when pushing to GitHub.

### [variables.tf](http://variables.tf) — Declaring Input Variables

Declare all variables your configuration needs. This file is safe to commit — it contains no values, just declarations:

> `# Authentication variables`
> 
> `variable "tenancy_ocid" {`
> 
>   `description = "OCID of your OCI tenancy"`
> 
>   `type        = string`
> 
> `}`
> 
> `variable "user_ocid" {`
> 
>   `description = "OCID of the OCI user for API authentication"`
> 
>   `type        = string`
> 
> `}`
> 
> `variable "fingerprint" {`
> 
>   `description = "Fingerprint of the API key"`
> 
>   `type        = string`
> 
> `}`
> 
> `variable "private_key_path" {`
> 
>   `description = "Local path to the OCI API private key (.pem)"`
> 
>   `type        = string`
> 
>   `default     = "~/.oci/oci_api_key.pem"`
> 
> `}`
> 
> `variable "region" {`
> 
>   `description = "OCI region identifier"`
> 
>   `type        = string`
> 
>   `default     = "ap-mumbai-1"`
> 
> `}`
> 
> `# Infrastructure variables`
> 
> `variable "compartment_ocid" {`
> 
>   `description = "OCID of the compartment to create resources in"`
> 
>   `type        = string`
> 
> `}`
> 
>  `variable "ssh_public_key" {`
> 
>   `description = "SSH public key to inject into the compute instance"`
> 
>   `type        = string`
> 
> `}`

### terraform.tfvars — Your Actual Values (Keep Private)

This file holds the real values for your variables. **Add it to .gitignore immediately:**

> `# terraform.tfvars — DO NOT COMMIT TO GIT`
> 
> `tenancy_ocid      = "ocid1.tenancy.oc1..aaaaaaaaxxxxxxxxxxx"`
> 
> `user_ocid         = "ocid1.user.oc1..aaaaaaaaxxxxxxxxxxx"`
> 
> `fingerprint       = "aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99"`
> 
> `private_key_path  = "/home/yourname/.oci/oci_api_key.pem"`
> 
> `region            = "ap-mumbai-1"`
> 
> `compartment_ocid  = "ocid1.compartment.oc1..aaaaaaaaxxxxxxxxxxx"`
> 
> `ssh_public_key   = "ssh-rsa AAAAB3NzaC1yc2E... your-public-key"`

### Create your .gitignore immediately:

> `# .gitignore`
> 
> `terraform.tfvars`
> 
> `*.tfvars`
> 
> `.terraform/`
> 
> `terraform.tfstate`
> 
> `terraform.tfstate.backup`
> 
> `*.pem`

## Section 4: Real Example — Build a Full OCI Network with Terraform

Now the exciting part. We'll create a [main.tf](http://main.tf) that provisions a complete, functional OCI network: a Virtual Cloud Network (VCN), a public subnet, an Internet Gateway, and the route table needed to make it all work.

### Understanding What We're Building

In OCI, before you can launch a compute instance, you need this networking chain in place:

•      **VCN** → The top-level private network, like an AWS VPC

•      **Subnet** → A subdivision of the VCN where resources live

•      **Internet Gateway** → Allows traffic between the subnet and the public internet

•      **Route Table** → Rules that direct traffic; the subnet must have a route pointing to the IGW

•      **Security List** → Stateful firewall rules (like AWS Security Groups) at the subnet level

### [main.tf](http://main.tf) — VCN, Subnet, Internet Gateway, and Security

> `# ── DATA SOURCE: Get Availability Domains ──────────────────────────`
> 
> `# This queries OCI to find the names of availability domains in your region.`
> 
> `# We use it to deploy the subnet into a specific AD.`
> 
> `data "oci_identity_availability_domains" "ads" {`
> 
>   `compartment_id = var.tenancy_ocid`
> 
> `}`
> 
> `# ── VIRTUAL CLOUD NETWORK (VCN) ─────────────────────────────────────`
> 
> `resource "oci_core_vcn" "main_vcn" {`
> 
>   `compartment_id = var.compartment_ocid`
> 
>   `cidr_block     = "10.0.0.0/16"`
> 
>   `display_name   = "terraform-demo-vcn"`
> 
>   `dns_label      = "tfdemov cn"   # Must be alphanumeric, max 15 chars`
> 
>   `freeform_tags = {`
> 
>     `"Project"     = "terraform-demo"`
> 
>     `"ManagedBy"   = "terraform"`
> 
>     `"Environment" = "learning"`
> 
>   `}`
> 
> `}`
> 
> `# ── INTERNET GATEWAY ─────────────────────────────────────────────────`
> 
> `# Required for instances in the public subnet to reach the internet.`
> 
> `resource "oci_core_internet_gateway" "igw" {`
> 
>   `compartment_id = var.compartment_ocid`
> 
>   `vcn_id         = oci_core_vcn.main_vcn.id   # Reference the VCN we just defined`
> 
>   `display_name   = "terraform-demo-igw"`
> 
>   `enabled        = true`
> 
> `}`
> 
> `# ── ROUTE TABLE ──────────────────────────────────────────────────────`
> 
> `# Directs all outbound internet traffic (0.0.0.0/0) through the IGW.`
> 
> `resource "oci_core_route_table" "public_rt" {`
> 
>   `compartment_id = var.compartment_ocid`
> 
>   `vcn_id         = oci_core_vcn.main_vcn.id`
> 
>   `display_name   = "public-route-table"`
> 
>   `route_rules {`
> 
>     `destination       = "0.0.0.0/0"           # All internet traffic`
> 
>     `network_entity_id = oci_core_internet_gateway.igw.id`
> 
>   `}`
> 
> `}`
> 
> `# ── SECURITY LIST ─────────────────────────────────────────────────────`
> 
> `# Defines firewall rules: allow SSH in, allow all traffic out.`
> 
> `resource "oci_core_security_list" "public_sl" {`
> 
>   `compartment_id = var.compartment_ocid`
> 
>   `vcn_id         = oci_core_vcn.main_vcn.id`
> 
>   `display_name   = "public-security-list"`
> 
>   `# Inbound: Allow SSH from anywhere`
> 
>   `ingress_security_rules {`
> 
>     `protocol  = "6"         # 6 = TCP`
> 
>     `source    = "0.0.0.0/0"`
> 
>     `stateless = false`
> 
>     `tcp_options {`
> 
>       `min = 22`
> 
>       `max = 22`
> 
>     `}`
> 
>     `description = "Allow SSH access"`
> 
>   `}`
> 
>   `# Inbound: Allow ICMP (ping)`
> 
>   `ingress_security_rules {`
> 
>     `protocol  = "1"         # 1 = ICMP`
> 
>     `source    = "0.0.0.0/0"`
> 
>     `stateless = false`
> 
>   `}`
> 
>   `# Outbound: Allow all traffic out`
> 
>   `egress_security_rules {`
> 
>     `protocol    = "all"`
> 
>     `destination = "0.0.0.0/0"`
> 
>     `stateless   = false`
> 
>   `}`
> 
> `}`
> 
> `# ── PUBLIC SUBNET ─────────────────────────────────────────────────────`
> 
> `resource "oci_core_subnet" "public_subnet" {`
> 
>   `compartment_id    = var.compartment_ocid`
> 
>   `vcn_id            = oci_core_vcn.main_vcn.id`
> 
>   `cidr_block        = "10.0.1.0/24"`
> 
>   `display_name      = "public-subnet"`
> 
>   `dns_label         = "public"`
> 
>   `# Attach the route table and security list we created above`
> 
>   `route_table_id    = oci_core_route_table.public_rt.id`
> 
>   `security_list_ids = [oci_core_security_list.public_sl.id]`
> 
>   `# false = allow public IPs on instances in this subnet`
> 
>   `prohibit_public_ip_on_vnic = false`
> 
> `}`

## Section 5: Adding a Compute Instance ([compute.tf](http://compute.tf))

Now let's add a compute instance to the public subnet. Create [compute.tf](http://compute.tf):

> `# ── LOOKUP: Find the Ubuntu 22.04 Image OCID ─────────────────────────`
> 
> `# OCI image OCIDs are region-specific. This data source finds the latest`
> 
> `# Ubuntu 22.04 image automatically — no hardcoding needed.`
> 
> `data "oci_core_images" "ubuntu_22" {`
> 
>   `compartment_id           = var.compartment_ocid`
> 
>   `operating_system         = "Canonical Ubuntu"`
> 
>   `operating_system_version = "22.04"`
> 
>   `shape                    = "VM.Standard.A1.Flex"   # ARM shape`
> 
>   `sort_by                  = "TIMECREATED"`
> 
>   `sort_order               = "DESC"`
> 
> `}`
> 
> `# ── COMPUTE INSTANCE ─────────────────────────────────────────────────`
> 
> `resource "oci_core_instance" "web_server" {`
> 
>   `availability_domain = data.oci_identity_availability_domains.ads.availability_domains[0].name`
> 
>   `compartment_id      = var.compartment_ocid`
> 
>   `display_name        = "terraform-demo-server"`
> 
>   `# Shape: ARM Always Free tier — 1 OCPU, 6GB RAM`
> 
>   `shape = "VM.Standard.A1.Flex"`
> 
>   `shape_config {`
> 
>     `ocpus         = 1`
> 
>     `memory_in_gbs = 6`
> 
>   `}`
> 
>   `# OS Image: Use the latest Ubuntu 22.04 found above`
> 
>   `source_details {`
> 
>     `source_type             = "image"`
> 
>     `source_id               = data.oci_core_images.ubuntu_22.images[0].id`
> 
>     `boot_volume_size_in_gbs = 50`
> 
>   `}`
> 
>   `# Network: Place in the public subnet we created`
> 
>   `create_vnic_details {`
> 
>     `subnet_id        = oci_core_subnet.public_subnet.id`
> 
>     `assign_public_ip = true    # Give this instance a public IP`
> 
>     `display_name     = "primary-vnic"`
> 
>   `}`
> 
>   `# SSH: Inject your public key so you can log in`
> 
>   `metadata = {`
> 
>     `ssh_authorized_keys = var.ssh_public_key`
> 
>   `}`
> 
>   `freeform_tags = {`
> 
>     `"ManagedBy"   = "terraform"`
> 
>     `"Environment" = "learning"`
> 
>   `}`
> 
> `}`

### [outputs.tf](http://outputs.tf) — See Your Results

Create [outputs.tf](http://outputs.tf) to display useful information after applying:

> `output "vcn_id" {`
> 
>   `description = "OCID of the created VCN"`
> 
>   `value       = oci_core_vcn.main_vcn.id`
> 
> `}`
> 
> `output "subnet_id" {`
> 
>   `description = "OCID of the public subnet"`
> 
>   `value       = oci_core_subnet.public_subnet.id`
> 
> `}`
> 
> `output "instance_public_ip" {`
> 
>   `description = "Public IP address of the compute instance"`
> 
>   `value       = oci_core_instance.web_server.public_ip`
> 
> `}`
> 
> `output "ssh_command" {`
> 
>   `description = "Ready-to-run SSH command to connect to your instance"`
> 
>   `value       = "ssh -i ~/.ssh/id_rsa ubuntu@${oci_core_instance.web_server.public_ip}"`
> 
> `}`

## Section 6: Running Terraform — Init, Plan, Apply

### terraform init

Initialize the project. This downloads the OCI provider plugin (~200MB) and sets up the working directory:

> `terraform init`
> 
> `# Expected output:`
> 
> `# Initializing the backend...`
> 
> `# Initializing provider plugins...`
> 
> `# - Finding oracle/oci versions matching ">= 6.0.0"...`
> 
> `# - Installing oracle/oci v6.x.x...`
> 
> `# Terraform has been successfully initialized!`

### terraform plan

Preview exactly what Terraform will create — nothing is changed in OCI yet:

> `terraform plan`
> 
> `# You'll see output like:`
> 
> `# Terraform will perform the following actions:`
> 
> `#`
> 
> `#   + resource "oci_core_vcn" "main_vcn" will be created`
> 
> `#   + resource "oci_core_internet_gateway" "igw" will be created`
> 
> `#   + resource "oci_core_route_table" "public_rt" will be created`
> 
> `#   + resource "oci_core_security_list" "public_sl" will be created`
> 
> `#   + resource "oci_core_subnet" "public_subnet" will be created`
> 
> `#   + resource "oci_core_instance" "web_server" will be created`
> 
> `#`
> 
> `# Plan: 6 to add, 0 to change, 0 to destroy.`

> **💡  Always Review the Plan**
> 
> The plan output is your change log before deployment. Read it carefully — especially lines starting with '-' (destroy) or '~' (modify in place). A '+' means create. Getting comfortable reading plans is the single most important Terraform habit.

### terraform apply

Apply the plan to create your real OCI resources. Terraform will pause and ask for confirmation:

> `terraform apply`
> 
> `# Terraform will show the plan again, then prompt:`
> 
> `# Do you want to perform these actions?`
> 
> `#   Terraform will perform the actions described above.`
> 
> `#   Only 'yes' will be accepted to approve.`
> 
> `#`
> 
> `# Enter a value: yes`
> 
> `# ... Terraform creates resources, then shows outputs:`
> 
> `# Outputs:`
> 
> `# instance_public_ip = "130.61.xxx.xxx"`
> 
> `# ssh_command = "ssh -i ~/.ssh/id_rsa ubuntu@130.61.xxx.xxx"`
> 
> `# subnet_id = "ocid1.subnet.oc1..."`
> 
> `# vcn_id = "ocid1.vcn.oc1..."`

Copy the ssh\_command output and connect to your new instance:

> `ssh -i ~/.ssh/id_rsa ubuntu@130.61.xxx.xxx`
> 
> `# You should be logged in to your brand new OCI VM!`
> 
> `# ubuntu@terraform-demo-server:~$`

## Section 7: Terraform Lifecycle — Modify and Destroy

### Making Changes

Terraform's real power is in how it handles changes. Edit your configuration — for example, add port 80 to the security list — then re-run terraform plan. Terraform shows exactly what will change, and terraform apply makes it so. **No manual clicking, no forgetting a step.**

> `# Add this inside the security list resource in main.tf:`
> 
>   `ingress_security_rules {`
> 
>     `protocol  = "6"`
> 
>     `source    = "0.0.0.0/0"`
> 
>     `stateless = false`
> 
>     `tcp_options { min = 80; max = 80 }`
> 
>     `description = "Allow HTTP access"`
> 
>   `}`

> `# Then:`
> 
> `terraform plan    # Shows: 1 to change`

> `terraform apply  # Updates the security list in place`

### Destroying Everything

When you're done experimenting, destroy all resources with a single command. This is one of Terraform's most powerful features — clean teardown with no orphaned resources:

> `terraform destroy`
> 
> `# Terraform shows everything it will delete, then prompts:`
> 
> `# Enter a value: yes`
> 
> `# All 6 resources are deleted cleanly.`
> 
> `# Destroy complete! Resources: 6 destroyed.`

> **⚠️  Destroy is Irreversible**
> 
> terraform destroy permanently deletes real OCI resources including any data on them. Always review the destroy plan carefully. For production environments, consider using Terraform's prevent\_destroy lifecycle argument on critical resources to add a safety net.

## Section 8: Best Practices for Terraform on OCI

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Practice</strong></p></td><td colspan="1" rowspan="1"><p><strong>Why It Matters</strong></p></td><td colspan="1" rowspan="1"><p><strong>How to Implement</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Use remote state</p></td><td colspan="1" rowspan="1"><p>Local state breaks in teams; gets lost if disk fails</p></td><td colspan="1" rowspan="1"><p>OCI Object Storage backend or Terraform Cloud</p></td></tr><tr><td colspan="1" rowspan="1"><p>Always run plan first</p></td><td colspan="1" rowspan="1"><p>Prevents surprises; builds discipline</p></td><td colspan="1" rowspan="1"><p>terraform plan -out=plan.out &amp;&amp; terraform apply plan.out</p></td></tr><tr><td colspan="1" rowspan="1"><p>Use consistent tagging</p></td><td colspan="1" rowspan="1"><p>Cost tracking, ownership, automation targeting</p></td><td colspan="1" rowspan="1"><p>Add freeform_tags to every resource</p></td></tr><tr><td colspan="1" rowspan="1"><p>Pin provider versions</p></td><td colspan="1" rowspan="1"><p>Prevents unexpected breaking changes on upgrade</p></td><td colspan="1" rowspan="1"><p>version = "&gt;= 6.0.0, &lt; 7.0.0"</p></td></tr><tr><td colspan="1" rowspan="1"><p>Use modules for patterns</p></td><td colspan="1" rowspan="1"><p>Avoid copy-pasting the same network config everywhere</p></td><td colspan="1" rowspan="1"><p>Create ./modules/vcn/ and call it from <a target="_self" rel="noopener noreferrer nofollow" class="text-primary underline underline-offset-2 hover:text-primary/80 cursor-pointer" href="http://main.tf" style="pointer-events: none;">main.tf</a></p></td></tr><tr><td colspan="1" rowspan="1"><p>Store secrets in OCI Vault</p></td><td colspan="1" rowspan="1"><p>Passwords in tfvars files are risky</p></td><td colspan="1" rowspan="1"><p>Reference OCI Vault secrets using data sources</p></td></tr><tr><td colspan="1" rowspan="1"><p>Use workspaces for environments</p></td><td colspan="1" rowspan="1"><p>Reuse same code for dev/staging/prod</p></td><td colspan="1" rowspan="1"><p>terraform workspace new staging</p></td></tr><tr><td colspan="1" rowspan="1"><p>Format and validate always</p></td><td colspan="1" rowspan="1"><p>Catches syntax errors before they reach plan</p></td><td colspan="1" rowspan="1"><p>terraform fmt &amp;&amp; terraform validate</p></td></tr></tbody></table>

## Bonus: Storing Terraform State in OCI Object Storage

For any serious usage — especially if you're working with a team or in a CI/CD pipeline — move your state file to OCI Object Storage. This prevents state file conflicts and ensures everyone works from the same source of truth.

> \# First, create an OCI bucket manually (or via Terraform in a separate root module)
> 
> \# Then update [provider.tf](http://provider.tf) to add a backend block:

> `terraform {`
> 
>   `required_providers {`
> 
>     `oci = { source = "oracle/oci", version = ">= 6.0.0" }`
> 
>   `}`
> 
>   `backend "http" {`
> 
>     `# OCI Object Storage pre-authenticated request URL for state`
> 
>     `address        = "https://objectstorage.ap-mumbai-1.oraclecloud.com/..."`
> 
>     `update_method  = "PUT"`
> 
>   `}`
> 
> `}`

> `# After updating the backend, re-initialize to migrate local state:`
> 
> `terraform init -migrate-state`

## Quick Reference: Common OCI Terraform Resources

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>OCI Resource</strong></p></td><td colspan="1" rowspan="1"><p><strong>Terraform Resource Type</strong></p></td><td colspan="1" rowspan="1"><p><strong>What It Creates</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Virtual Cloud Network</p></td><td colspan="1" rowspan="1"><p>oci_core_vcn</p></td><td colspan="1" rowspan="1"><p>Private network for your resources</p></td></tr><tr><td colspan="1" rowspan="1"><p>Subnet</p></td><td colspan="1" rowspan="1"><p>oci_core_subnet</p></td><td colspan="1" rowspan="1"><p>Sub-division of VCN for resource placement</p></td></tr><tr><td colspan="1" rowspan="1"><p>Internet Gateway</p></td><td colspan="1" rowspan="1"><p>oci_core_internet_gateway</p></td><td colspan="1" rowspan="1"><p>Public internet access for a VCN</p></td></tr><tr><td colspan="1" rowspan="1"><p>NAT Gateway</p></td><td colspan="1" rowspan="1"><p>oci_core_nat_gateway</p></td><td colspan="1" rowspan="1"><p>Outbound-only internet for private subnets</p></td></tr><tr><td colspan="1" rowspan="1"><p>Route Table</p></td><td colspan="1" rowspan="1"><p>oci_core_route_table</p></td><td colspan="1" rowspan="1"><p>Traffic routing rules</p></td></tr><tr><td colspan="1" rowspan="1"><p>Security List</p></td><td colspan="1" rowspan="1"><p>oci_core_security_list</p></td><td colspan="1" rowspan="1"><p>Subnet-level firewall rules</p></td></tr><tr><td colspan="1" rowspan="1"><p>Network Security Group</p></td><td colspan="1" rowspan="1"><p>oci_core_network_security_group</p></td><td colspan="1" rowspan="1"><p>Instance-level firewall rules</p></td></tr><tr><td colspan="1" rowspan="1"><p>Compute Instance</p></td><td colspan="1" rowspan="1"><p>oci_core_instance</p></td><td colspan="1" rowspan="1"><p>Virtual machine</p></td></tr><tr><td colspan="1" rowspan="1"><p>Block Volume</p></td><td colspan="1" rowspan="1"><p>oci_core_volume</p></td><td colspan="1" rowspan="1"><p>Persistent block storage disk</p></td></tr><tr><td colspan="1" rowspan="1"><p>Object Storage Bucket</p></td><td colspan="1" rowspan="1"><p>oci_objectstorage_bucket</p></td><td colspan="1" rowspan="1"><p>S3-compatible object storage</p></td></tr><tr><td colspan="1" rowspan="1"><p>Autonomous Database</p></td><td colspan="1" rowspan="1"><p>oci_database_autonomous_database</p></td><td colspan="1" rowspan="1"><p>Managed Oracle DB</p></td></tr><tr><td colspan="1" rowspan="1"><p>OKE Cluster</p></td><td colspan="1" rowspan="1"><p>oci_containerengine_cluster</p></td><td colspan="1" rowspan="1"><p>Managed Kubernetes cluster</p></td></tr><tr><td colspan="1" rowspan="1"><p>Load Balancer</p></td><td colspan="1" rowspan="1"><p>oci_load_balancer_load_balancer</p></td><td colspan="1" rowspan="1"><p>HTTP/TCP load balancer</p></td></tr><tr><td colspan="1" rowspan="1"><p>IAM Compartment</p></td><td colspan="1" rowspan="1"><p>oci_identity_compartment</p></td><td colspan="1" rowspan="1"><p>Resource organization unit</p></td></tr></tbody></table>

* * *

***Safe Harbour:*** *This site may contain forward-looking statements regarding cloud architectures, infrastructure strategies, product roadmaps, performance benchmarks, or emerging technologies that are subject to change without notice. Actual results may vary due to technical, operational, or market factors. All content is provided for informational and educational purposes only and does not constitute professional, legal, financial, or investment advice. We undertake no obligation to update such statements.*