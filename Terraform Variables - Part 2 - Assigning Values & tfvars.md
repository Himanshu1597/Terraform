# Terraform Variables — Part 2: Assigning Values & tfvars

> This continues from Part 1. By now you've declared variables in `variables.tf` and used them in `main.tf` with `var.`. The next step: actually giving those variables their values.

---

## 1. Ways to Assign Values to Variables

There are **3 common methods** (plus one extra worth knowing):

---

### Method 1: Inline CLI (`-var` flag)

Pass values directly in the terminal when running a command.

```bash
terraform apply -var="vpc_cidr=10.0.0.0/16" -var="vpc_name=dev-vpc"
```

- Good for **quick, one-off testing**
- Not recommended for production — values aren't saved anywhere, easy to make mistakes
- Becomes unwieldy with many variables

---

### Method 2: `terraform.tfvars` File (Most Common)

Create a file named exactly `terraform.tfvars` in your project root and write key-value pairs in it.

```hcl
# terraform.tfvars
vpc_cidr = "10.0.0.0/16"
vpc_name = "dev-vpc"
```

- Terraform **automatically loads** this file when you run `terraform plan` or `terraform apply` — no flags needed
- This is the standard, most widely used approach

**Full example:**

```
project/
├── main.tf
├── variables.tf
└── terraform.tfvars
```

`main.tf`
```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "my_vpc" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "MyVPC"
  }
}
```

`variables.tf`
```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}
```

`terraform.tfvars`
```hcl
vpc_cidr = "10.0.0.0/16"
```

---

### Method 3: Custom Named `.tfvars` Files (`-var-file`)

Create separate files like `dev.tfvars`, `prod.tfvars` and pass them explicitly with a flag.

```bash
terraform apply -var-file="dev.tfvars"
terraform apply -var-file="prod.tfvars"
```

- These files are **not auto-loaded** — you must pass them manually
- Ideal for managing multiple environments from one codebase

---

### Method 4: Environment Variables (`TF_VAR_`) — Worth Knowing

Terraform reads environment variables prefixed with `TF_VAR_`.

```bash
export TF_VAR_vpc_cidr="10.0.0.0/16"
export TF_VAR_vpc_name="dev-vpc"

terraform apply
```

- Useful in **CI/CD pipelines** where you don't want variable values written to files
- Good for secrets — set them in the pipeline environment, not in committed files

---

### Method 5: Module Inputs

When calling a reusable Terraform module, you pass variable values directly in the module block. This is covered in the Modules section.

---

## 2. How Terraform Uses Variable Values During `plan` and `apply`

Once you've declared variables and assigned values, here's exactly what Terraform does when you run `plan` or `apply`:

### The Execution Flow

```
Step 1 — Read all .tf files in the working directory
         (main.tf, variables.tf, .tfvars files, etc.)

Step 2 — Match every var.x in your code to:
         → Its declaration in variables.tf
         → Its assigned value (from .tfvars, CLI flag, or module input)

Step 3 — Build a full dependency graph using the resolved values
         (Terraform figures out what depends on what)

Step 4 — During plan: show what will be created/updated/deleted
         using the actual resolved values

Step 5 — During apply: provision resources using those values
         (e.g., create a VPC with the CIDR from your variable)
```

### Key Rule

> **Terraform always resolves ALL variables before taking any action.**
>
> This is why you must declare variables in `variables.tf` and assign their values before running `plan` or `apply`. If a required variable is missing a value, Terraform stops and asks for it.

---

## 3. Multi-file Variable Configuration

### What It Is

Instead of editing the same `terraform.tfvars` file every time you deploy to a different environment, you create **separate `.tfvars` files** — one per environment or concern.

```
project/
├── main.tf
├── variables.tf
├── dev.tfvars
├── prod.tfvars
└── staging.tfvars
```

### The Problem It Solves

Without this, you'd have to:
1. Open `terraform.tfvars`
2. Manually change the values
3. Save the file
4. Run apply
5. Then change it back for the next environment

This is repetitive and error-prone. Multi-file configuration eliminates that.

### How It Works

`dev.tfvars`
```hcl
vpc_cidr = "192.168.10.0/24"
vpc_name = "dev-vpc"
```

`prod.tfvars`
```hcl
vpc_cidr = "192.168.20.0/24"
vpc_name = "prod-vpc"
```

Deploy to Dev:
```bash
terraform apply -var-file="dev.tfvars"
```

Deploy to Prod:
```bash
terraform apply -var-file="prod.tfvars"
```

Same `main.tf`, zero edits — just switch the file.

---

### Use Cases

| Use Case | What to Do |
|---|---|
| **Environment separation** | `dev.tfvars`, `staging.tfvars`, `prod.tfvars` |
| **Split by component/team** | `network.tfvars`, `security.tfvars`, `tags.tfvars` |
| **Sensitive data** | `secrets.tfvars` — stored securely, never committed to Git |
| **CI/CD automation** | Pass the right `-var-file` based on branch or pipeline stage |

> In large teams, each team (network, security, DevOps) can own and manage their own `.tfvars` file independently.

---

### Important: What Gets Auto-loaded vs What Doesn't

| File | Auto-loaded? |
|---|---|
| `terraform.tfvars` | Yes — always |
| `*.auto.tfvars` | Yes — always |
| `dev.tfvars`, `prod.tfvars`, etc. | **No** — must use `-var-file` |

This is intentional. It gives you **explicit control** over which environment config gets applied.

---

## 4. `.auto.tfvars` Files

### What It Is

Any file ending in `.auto.tfvars` is **automatically loaded** by Terraform — no `-var-file` flag needed. Think of it as an extension of `terraform.tfvars` that you can split into multiple files.

```
# These are all auto-loaded:
cidr.auto.tfvars
name.auto.tfvars
network.auto.tfvars
```

### Why Use It

- No need to type `-var-file=...` every time
- Great for **automation and CI/CD scripting** where you want things to just work
- Lets you split variable files by purpose without any extra commands

### Example

```
terraform-vpc/
├── main.tf
├── variables.tf
├── cidr.auto.tfvars
└── name.auto.tfvars
```

`variables.tf`
```hcl
variable "vpc_cidr" {}
variable "vpc_name" {}
```

`cidr.auto.tfvars`
```hcl
vpc_cidr = "10.0.0.0/16"
```

`name.auto.tfvars`
```hcl
vpc_name = "My-Auto-VPC"
```

Run:
```bash
terraform init
terraform apply
```

Terraform auto-loads both files and uses both values — no flags required.

---

### `.auto.tfvars` vs Custom `.tfvars` — When to Use Which

| | `.auto.tfvars` | `dev.tfvars` / `prod.tfvars` |
|---|---|---|
| Auto-loaded | Yes | No — needs `-var-file` |
| Best for | Stable config, automation, splitting by concern | Environment switching, explicit control |
| Control | Less explicit | More explicit |

---

## 5. Variable Precedence (Critical — Not in PDFs)

When the same variable gets a value from multiple sources, Terraform follows a strict **priority order** — the higher the source in this list, the higher its priority:

```
1. Default value in variables.tf           ← lowest priority
2. terraform.tfvars (auto-loaded)
3. *.auto.tfvars (auto-loaded, alphabetical order)
4. -var-file="..." CLI flag
5. -var="..." CLI flag                     ← highest priority
```

**What this means practically:**

- A value passed via `-var` on the CLI **always wins**, overriding everything else
- `terraform.tfvars` overrides the `default` in `variables.tf`
- `*.auto.tfvars` overrides `terraform.tfvars`

**Example:** If `vpc_cidr = "10.0.0.0/16"` is the default, but `terraform.tfvars` has `vpc_cidr = "172.16.0.0/16"`, and you also run with `-var="vpc_cidr=192.168.0.0/16"` — Terraform uses `192.168.0.0/16`.

---

## 6. Quick Reference: All the Files and Their Roles

| File | Role | Auto-loaded? |
|---|---|---|
| `main.tf` | Resource definitions using `var.` | — |
| `variables.tf` | Variable declarations (name, type, default) | — |
| `terraform.tfvars` | Default value assignments | Yes |
| `*.auto.tfvars` | Auto-loaded value assignments, split by purpose | Yes |
| `dev.tfvars`, `prod.tfvars` | Environment-specific values | No — use `-var-file` |
| `secrets.tfvars` | Sensitive values (keep out of Git) | No — use `-var-file` |

---

## Practice Exercises

### Exercise 1 — Create and Use terraform.tfvars
Write a `variables.tf` with `instance_type` and `ami_id` (both required, no defaults). Then create a `terraform.tfvars` that provides values for both. What happens if you delete `terraform.tfvars` and run `terraform plan`?

---

### Exercise 2 — Multi-Environment Setup
You manage infrastructure for Dev and Prod. Create:
- `dev.tfvars` with `env = "dev"` and `vpc_cidr = "10.0.0.0/16"`
- `prod.tfvars` with `env = "prod"` and `vpc_cidr = "172.16.0.0/16"`

Write the command to deploy each environment without touching `main.tf`.

---

### Exercise 3 — auto.tfvars Split by Purpose
Create a project with these auto-loading files:
- `network.auto.tfvars` → holds `vpc_cidr`
- `tags.auto.tfvars` → holds `vpc_name` and `environment`

Write all three files and confirm Terraform loads them all with just `terraform apply`.

---

### Exercise 4 — Precedence Test
Given this setup:
- `variables.tf` has `default = "t2.micro"` for `instance_type`
- `terraform.tfvars` has `instance_type = "t2.small"`
- You run: `terraform apply -var="instance_type=t3.medium"`

What value does Terraform use? Explain why.

---

### Exercise 5 — Sensitive Data Separation
You have a database password that must not be committed to Git. Write:
1. The variable declaration in `variables.tf` (with `sensitive = true`, no default)
2. A `secrets.tfvars` file with the value
3. The correct `terraform apply` command to use it
4. What file should be added to `.gitignore` and why?

---

### Exercise 6 — Spot the Difference
Two developers set up variables differently:

**Dev A:**
```
terraform apply -var="vpc_cidr=10.0.0.0/16" -var="vpc_name=dev" -var="region=us-east-1" -var="env=dev"
```

**Dev B:**
```
# dev.tfvars
vpc_cidr = "10.0.0.0/16"
vpc_name = "dev"
region   = "us-east-1"
env      = "dev"

terraform apply -var-file="dev.tfvars"
```

Both work. Which approach is better for team use and why? What problem does Dev A's approach create at scale?

---

*Notes prepared from: Milestone 5 — PDFs 3, 4, 5, 6*
