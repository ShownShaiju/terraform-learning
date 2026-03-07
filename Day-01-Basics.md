# Terraform Notes Day 1

These notes capture **how I understood Terraform step by step**, including the questions that came to my mind while learning. The idea is to keep the thinking process visible instead of just dumping theory.

---

# What Problem Terraform Solves

When deploying infrastructure manually in a cloud console (like AWS), engineers often:

* click through dozens of settings
* forget configurations
* cannot reproduce the setup easily
* cannot version control infrastructure

Terraform solves this by letting you **describe infrastructure using code**.

This concept is called **Infrastructure as Code (IaC)**.

Example Terraform configuration:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0abcdef"
  instance_type = "t2.micro"
}
```

Instead of manually creating servers, Terraform reads the configuration and creates the infrastructure.

---

# Question I Had

**"But normally you create an AMI first and get the AMI ID. How does Terraform work here? Isn't that backwards?"**

Answer:

Terraform is **not creating the AMI** in this example.

The AMI already exists (for example Amazon Linux or Ubuntu). Terraform simply **launches an EC2 instance using that AMI**.

Think of an AMI as a **template image of a machine**.

Terraform uses the template to create servers.

---

# Terraform Workflow

Terraform always follows the same workflow.

1. Write configuration files (`.tf` files)
2. Initialize Terraform
3. Preview infrastructure changes
4. Apply the changes

Commands:

```
terraform init
terraform plan
terraform apply
terraform destroy
```

Explanation:

* **init** → downloads required provider plugins
* **plan** → shows what Terraform will create or change
* **apply** → actually creates the infrastructure
* **destroy** → removes infrastructure

---

# Question I Had

**"When we run terraform init, is it like git init? Do we run it before writing the tf files or after?"**

Answer:

The workflow usually looks like this:

```
1. Create project folder
2. Write .tf files
3. terraform init
4. terraform plan
5. terraform apply
```

So yes, the idea is somewhat similar to `git init`, but the purpose is different.

`terraform init` prepares the working directory and **downloads the provider plugins Terraform needs to talk to cloud APIs**.

---

# Providers

Providers are plugins that allow Terraform to communicate with external APIs.

Examples:

* AWS
* Azure
* Google Cloud
* Kubernetes
* Docker

Example:

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

When running `terraform init`, Terraform downloads the provider plugin.

---

# Question I Had

**"By provider do you mean AWS itself?"**

Yes.

The provider acts as the **bridge between Terraform and the cloud platform's API**.

Example:

Terraform → AWS Provider → AWS API → Infrastructure

---

# Resources

Resources represent the actual infrastructure Terraform creates.

Example:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0abcdef"
  instance_type = "t2.micro"
}
```

Here Terraform creates an **EC2 instance**.

---

# Outputs

Outputs display information about created infrastructure after deployment.

Example:

```hcl
output "instance_ip" {
  value = aws_instance.web_server.public_ip
}
```

---

# Question I Had

**"Does the instance public IP get stored in a variable called instance_ip?"**

Not exactly.

Terraform reads the value from the **state file** and prints it after `terraform apply`.

Outputs are mainly used to **expose useful information after infrastructure creation**.

---

# Terraform State

Terraform keeps track of infrastructure using a **state file**.

```
terraform.tfstate
```

The state file contains:

* resource IDs
* metadata
* dependency information
* current infrastructure snapshot

Terraform compares:

```
Desired state (configuration)
vs
Current state (state file)
```

This comparison generates the execution plan.

---

# Question I Had

**"Why store Terraform state in S3 instead of just pushing it to GitHub?"**

Two main reasons:

### 1. Sensitive Data

State files can contain:

* passwords
* database connection strings
* private infrastructure details

So storing them publicly is risky.

### 2. State Locking

In teams, multiple engineers may run Terraform at the same time.

Without locking, this can corrupt the state.

Production setups usually use:

```
S3 → store state
DynamoDB → state locking
```

---

# Follow-up Question I Had

**"But GitHub has private repositories. So why not store the Terraform state there instead? And is there no way to implement state locking using GitHub?"**

Answer:

Even with private repositories, GitHub is still **not a good place for Terraform state**.

Reasons:

### 1. GitHub Does Not Provide Native State Locking

Terraform needs **exclusive locking** so that only one engineer can modify infrastructure at a time.

Example scenario:

```
Engineer A runs terraform apply
↓
State must be locked
↓
Engineer B must wait until the operation finishes
```

Backends like **S3 + DynamoDB** support this locking mechanism.

GitHub does not provide a locking system Terraform can use.

### 2. Git Is Not Designed for Rapidly Changing State

Terraform state updates **every time infrastructure changes**.

If stored in GitHub, the repository would constantly receive commits like:

```
apply
commit state
apply
commit state
apply
commit state
```

This creates unnecessary commit noise and potential merge conflicts.

### 3. Terraform Needs Fast Read/Write Access

Terraform frequently reads and updates the state during operations.

Git workflows involve:

```
pull
modify
commit
push
```

This makes Git unsuitable for managing Terraform state safely.

Because of these reasons, teams normally store:

```
Terraform code → GitHub
Terraform state → S3 / Terraform Cloud
State locking → DynamoDB
```

---

# Remote State

Teams usually store the Terraform state remotely.

Example configuration:

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state-prod"
    key    = "infra/terraform.tfstate"
    region = "ap-south-1"
  }
}
````

Explanation:

* `backend "s3"` → tells Terraform to store state in S3
* `bucket` → S3 bucket name
* `key` → path of the state file inside the bucket
* `region` → region where the bucket exists

---

# Question I Had

**"Why does this code use backend instead of resource aws_s3_bucket?"**

Because **backend is not infrastructure**.

It is just Terraform configuration telling Terraform **where to store its state**.

Resources are what actually create infrastructure.

---

# Terraform File Structure

Terraform loads all files that end with `.tf`.

These filenames are **not special keywords**, just conventions.

Common project structure:

```
main.tf
variables.tf
outputs.tf
providers.tf
backend.tf
```

Terraform merges all `.tf` files internally.

So these would all work the same:

```
network.tf
compute.tf
banana.tf
```

As long as they end with `.tf`.

---

# Question I Had

**"Are these filenames required or can I write whatever.tf? Are they case sensitive?"**

You can name them anything.

Terraform only requires the **`.tf` extension**.

However, variables and identifiers **are case sensitive**.

Example:

```
instance_type
Instance_Type
```

Terraform treats these as different values.

---

# JSON Terraform Config

Terraform can also read configuration in JSON format.

Example file:

```
main.tf.json
```

Example configuration:

```json
{
  "resource": {
    "aws_instance": {
      "web": {
        "instance_type": "t2.micro"
      }
    }
  }
}
```

---

# Question I Had

**"Why would anyone use .tf.json instead of .tf?"**

Usually humans don't.

This format is mainly used when **other tools generate Terraform configurations automatically**.

Humans normally write Terraform using **HCL (.tf files)**.

---