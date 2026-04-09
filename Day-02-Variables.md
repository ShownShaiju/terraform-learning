# Terraform Notes Day 2
*Variables, Locals, and Environments*

These notes build on Day 1. Same idea — breaking things down to first principles, keeping the questions visible because the questions are where the actual understanding lives.

---

## Correction From Day 1

In Day 1 I wrote that `terraform init` downloads provider plugins and prepares the working directory. That part is correct.

But I had a wrong mental model about *when* Terraform actually talks to AWS and *when* the state file gets touched. I thought `terraform init` was doing some kind of comparison or state check. It is not.

Here is the corrected understanding:

`terraform init` never contacts AWS. It never reads or writes the state file. It is purely local. It downloads the provider plugin binary into a `.terraform/` folder in your project. That is all.

The actual comparison — where Terraform checks what you want against what exists — happens during `terraform plan`. Not before.

And the state file only gets written during `terraform apply` and `terraform destroy`. Those are the only two commands that modify `terraform.tfstate`.

The corrected mental model for the full workflow:

```
terraform init    →  downloads plugins locally
                     zero AWS contact
                     zero state file changes

terraform plan    →  reads your .tf files
                     reads terraform.tfstate
                     calls AWS API in read-only mode
                     compares all three
                     shows you the diff
                     still changes nothing

terraform apply   →  makes real changes in AWS
                     then writes to terraform.tfstate

terraform destroy →  destroys real infrastructure
                     then updates terraform.tfstate
```

One more thing worth adding here. During `terraform plan`, the comparison between the state file and the live AWS API can reveal something called **state drift**. This happens when someone manually changed or deleted something in AWS directly — through the console or CLI — without going through Terraform. The state file still says that resource exists exactly as Terraform last knew it. But AWS reality is different now. `terraform plan` catches this because it cross checks the state file against the live API. This is exactly why you should never manually change infrastructure that Terraform manages. It breaks the trust between the state file and reality.

---

## Why Variables Exist

By the end of Day 1 the Terraform code I was writing looked like this:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0abcdef"
  instance_type = "t2.micro"
}
```

This works. But every value is hardcoded directly into the resource. If I wanted to deploy the same infrastructure for staging with a bigger instance type, or for production in a different region, I would have to go find every hardcoded value across potentially dozens of files and change them manually.

That is not sustainable. Variables solve this by separating the *structure* of the infrastructure from the *values* that configure it.

The structure stays the same everywhere. Only the values change per environment.

---

## The Three File System

Variables in Terraform are not one thing. They are a system of three files working together. This is the part that confused me at first because I kept thinking of it as a single concept.

**`variables.tf` is the contract.**

This file declares what inputs the project accepts. Think of it as defining the parameters a function accepts before you write the function body. You are not setting values here. You are saying these inputs exist, here is their type, and here is their default if no value is provided.

```hcl
variable "instance_type" {
  description = "The size of the EC2 instance"
  type        = string
  default     = "t2.micro"
}

variable "environment" {
  description = "Which environment this is for"
  type        = string
}
```

Notice `environment` has no default. That means it is required. Terraform will refuse to run without it. `instance_type` has a default so it is optional — if nothing is provided Terraform uses `"t2.micro"` automatically.

**`terraform.tfvars` is where actual values live.**

```hcl
instance_type = "t3.small"
environment   = "production"
```

This is the file that changes between environments. The `.tf` files stay identical. Only the values in `.tfvars` change.

**`main.tf` is where variables get used, with `var.`**

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0abcdef"
  instance_type = var.instance_type

  tags = {
    Environment = var.environment
  }
}
```

The syntax is always `var.` followed by the variable name. Nothing else.

---

## Question I Had

**"In the tags block, what is `Environment`? Is that a Terraform keyword?"**

No. `Environment` on the left side of the tags map is just a tag key — a label I am choosing to attach to the AWS resource. I could name it anything. `Environment`, `Env`, `Stage`, `Banana`. AWS does not care. It is just metadata that shows up in the AWS console so I can filter and identify resources later.

`var.environment` on the right side is the Terraform variable — the actual value coming from the `.tfvars` file.

After apply this tag appears in the AWS console as:
```
Environment : production
```

It has no functional impact on the infrastructure. It is purely for human organisation.

---

## The Function Analogy

The cleanest way I found to understand the three file system is to map it directly to a function in any programming language.

```python
# Python
def create_server(instance_type, environment):
    # do something
    pass

create_server(instance_type="t3.small", environment="production")
```

Map that to Terraform:

```
variables.tf     =  defining the function parameters
main.tf          =  the function body using those parameters
terraform.tfvars =  calling the function with specific values
```

The `variables.tf` and `main.tf` together are the reusable function. The `.tfvars` file is what changes every time you call it for a different environment.

---

## Variable Types

Variables can enforce types. If I declare a variable as `type = number` and someone passes `"hello"` as the value, Terraform throws an error immediately during `plan` — before anything touches AWS. This is especially valuable in teams.

```hcl
# String — plain text
variable "region" {
  type    = string
  default = "ap-south-1"
}

# Number — for ports, counts, sizes
variable "instance_count" {
  type    = number
  default = 2
}

# Boolean — true or false flags
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List — ordered collection of same type
variable "availability_zones" {
  type    = list(string)
  default = ["ap-south-1a", "ap-south-1b"]
}

# Map — key value pairs
variable "tags" {
  type = map(string)
  default = {
    Team    = "backend"
    Project = "devRPG"
  }
}
```

---

## The Four Ways To Pass Variable Values

This surprised me. There is not just one way to pass values to variables. There are four, and they have a priority order.

**Way 1 — `terraform.tfvars` auto-loaded**

If a file named exactly `terraform.tfvars` exists in the project folder, Terraform loads it automatically without being told to. No extra flags needed.

**Way 2 — Named tfvars file passed manually**

```bash
terraform apply -var-file="production.tfvars"
```

The filename here means nothing to Terraform. It reads whatever file I point it to. I could name it `v2.tfvars` or `banana.tfvars` — it does not matter. The only magic filename is `terraform.tfvars` which gets auto-loaded.

**Way 3 — Inline on the command line**

```bash
terraform apply -var="environment=production"
```

Useful for quick one-off overrides without editing any file.

**Way 4 — Environment variables**

```bash
export TF_VAR_environment="production"
terraform apply
```

Terraform automatically reads any environment variable starting with `TF_VAR_` and maps it to the matching variable name. This is how CI/CD pipelines like GitHub Actions pass secrets to Terraform without storing them in any file.

**Priority order when multiple sources provide the same variable:**

```
Highest → -var flag on command line
          -var-file flag
          *.auto.tfvars files
          terraform.tfvars
Lowest  → default value in variables.tf
```

If the same variable is set in multiple places, the highest priority wins.

---

## Question I Had

**"What happens if a required variable — one with no default — has no value provided from any of these four ways?"**

Terraform does not silently use null or an empty string. It refuses to run at all and prints a hard error during `plan` before anything touches AWS:

```
│ Error: No value for required variable
│
│ The root module input variable "environment" is not set,
│ and has no default value. Use a -var or -var-file command
│ line argument to provide a value for this variable.
```

Failing loudly early is the right behaviour. Better to crash at `plan` than to silently deploy wrong infrastructure.

The full decision table for how Terraform resolves variable values:

```
Has default + value provided     →  uses provided value
Has default + no value provided  →  uses default
No default  + value provided     →  uses provided value
No default  + no value provided  →  hard error at plan stage
```

---

## Locals — Internal Computed Values

Locals are similar to variables but with one fundamental difference. Variables are inputs — values that come from outside the code, from `.tfvars` files or the command line. Locals are internal — they are defined inside the code and computed from other values. Nobody passes them in from outside.

`locals {}` is not a separate file. It is a block you write inside any `.tf` file. Most people put it in `main.tf` or in a dedicated `locals.tf` if it grows large.

```hcl
locals {
  app_name  = "devRPG"
  full_name = "${local.app_name}-${var.environment}"
}
```

The `${}` is string interpolation — building a new string from existing values. If `var.environment` is `"prod"` then `local.full_name` becomes `"devRPG-prod"` automatically.

You reference locals with `local.` — singular, not `locals.`:

```hcl
resource "aws_s3_bucket" "app" {
  bucket = local.full_name
}
```

---

## Question I Had

**"Why would I use a local instead of just using the variable directly everywhere?"**

Two reasons.

First, DRY — Don't Repeat Yourself. If the same computed value is used in ten different resources, defining it once as a local means changing it in one place instead of ten.

Second, locals can compute values that variables cannot. Variables are just raw inputs. Locals can combine multiple variables, transform them, or build derived values.

Example of where a local is genuinely useful:

```hcl
locals {
  # Build a consistent name used across many resources
  full_name = "${var.app_name}-${var.environment}-${var.region}"
  
  # Decide instance size based on environment
  is_prod = var.environment == "production"
}
```

Now `local.full_name` can be referenced in every resource that needs a consistent naming convention, and if the naming pattern ever changes you update it in exactly one place.

---

## When To Use Variable vs Local

```
Value comes from outside (different per environment)  →  variable
Value is computed from other values                   →  local
Value is used in many places but never changes        →  local
You want someone else to be able to override it       →  variable
It is a secret passed in from CI/CD                   →  variable via TF_VAR_
```

---

## Understanding Environments (Dev, Staging, Production)

This came up naturally while learning variables because the whole point of variables is deploying the same code to multiple environments with different values.

There are three environments most teams use:

**Local / Dev** — your personal sandbox. Runs against your own AWS account. Only you use it. Break things freely here, nobody cares. This is where you run `terraform apply` from WSL right now.

**Staging** — a shared environment that mirrors production as closely as possible but real users never touch it. The whole team deploys here first to test that everything works together before it goes live. Think of it as a dress rehearsal. Same infrastructure setup as production, test data inside it.

**Production** — real users, real data, real consequences.

Staging exists because things that work perfectly on your laptop often break in a production-like environment. Staging catches those surprises before your actual users do.

In Terraform terms you would have three `.tfvars` files:

```
dev.tfvars         →  small instances, cheap setup, your personal sandbox
staging.tfvars     →  mirrors production, shared team environment
production.tfvars  →  real infrastructure, real users
```

The same `.tf` code deploys all three. Only the values differ.

---

## What I Know Now That I Did Not Know After Day 1

```
- terraform init never touches AWS or the state file
- terraform plan is the command that does the three-way comparison
- State drift — what it is and why manual changes cause it
- Variables — declaring, using, and passing values in four ways
- The three file system — variables.tf, terraform.tfvars, main.tf
- Variable types — string, number, bool, list, map
- Required vs optional variables and what happens when values are missing
- Locals — what they are, where they live, when to use them vs variables
- Dev, Staging, Production — what each environment is and why staging exists
- terraform.tfvars is the one auto-loaded magic filename
```