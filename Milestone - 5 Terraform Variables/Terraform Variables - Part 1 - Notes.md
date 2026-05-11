# Terraform Variables

---

## 1. What Are Terraform Variables?

Think of a variable as a **named container** that holds a value — like a box with a label on it.

Instead of writing the same value (like a CIDR block, region, or name) over and over inside your Terraform files, you define it **once** as a variable and **refer to it** wherever needed.

> **Simple Definition:** A Terraform variable stores a value (like region, instance type, name, etc.) that you can reuse across your Terraform configuration files.

### Why Variables Exist

| Problem Without Variables | Solution With Variables |
|---|---|
| Values are hardcoded everywhere | Values are defined in one place |
| Changing a value means editing every file | Change it in one place, reflects everywhere |
| You can't reuse the same code for Dev/QA/Prod | Same code works for all environments with different inputs |
| Easy to make mistakes during copy-paste | Clean, consistent, less error-prone |

### Benefits at a Glance

- **Reusable** — Write once, use across many environments
- **Clean** — No duplicate values scattered in code
- **Easy to change** — Update one variable, everything updates

---

## 2. The Problem Variables Solve — A Real Example

### Without Variables (Hardcoded — Bad Practice)

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "dev-vpc"
  }
}
```

**Problems:**
- You cannot reuse this for QA, Stage, or Prod environments
- Every time you want to change the CIDR or name, you have to **manually find and edit** it
- If this value appears in 10 places, you must change it 10 times — risky!

---

### With Variables (Dynamic — Good Practice)

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = var.vpc_name
  }
}
```

**Benefits:**
- Reuse the same VPC logic for any environment — just pass different values
- The resource block itself **never needs to be touched** again
- Clean separation between **logic** (what to create) and **data** (what values to use)

---

## 3. How to Declare Variables

Variables are declared using the `variable` block — typically in a file called `variables.tf`.

### Basic Syntax (Minimal Declaration)

```hcl
variable "variable_name" {}
```

This is the bare minimum. It tells Terraform: *"Hey, I'm going to use a variable called this — it exists."*

### Full Syntax (With All Properties)

```hcl
variable "variable_name" {
  description = "What this variable is for"
  type        = string
  default     = "some-default-value"
  sensitive   = false

  validation {
    condition     = length(var.variable_name) > 0
    error_message = "Value must not be empty."
  }
}
```

### Variable Properties Explained

| Property | What It Does | Required? |
|---|---|---|
| `description` | Human-readable explanation of the variable | No, but recommended |
| `type` | Data type: `string`, `number`, `bool`, `list`, `map`, `object` | No |
| `default` | Fallback value if no value is provided | No |
| `sensitive` | Hides the value from logs and terminal output (for passwords, tokens) | No |
| `validation` | Custom rule to validate the input value | No |

---

## 4. Two Types of Variables — Optional vs Required

### Optional Variable (Has a Default)

If a variable has a `default` value, Terraform will use that value automatically if you don't provide one.

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}
```

- You **don't have to** pass a value when running `terraform apply`
- Terraform quietly uses `10.0.0.0/16`

---

### Required Variable (No Default)

If a variable has **no default**, Terraform will **stop and ask you** for a value at runtime.

```hcl
variable "vpc_name" {}
```

- Terraform will prompt: *"var.vpc_name: Enter a value:"*
- You **must** provide a value, or the run will fail

---

## 5. How to Use Variables

Once declared, you reference a variable in your `.tf` files using the `var.` prefix.

### Syntax

```hcl
var.variable_name
```

### Rules to Follow

1. **Always use the `var.` prefix** when referencing a variable inside any `.tf` file
2. **Declare the variable** in `variables.tf` before running `terraform plan` or `terraform apply`

---

## 6. Complete Working Example

This example shows the full workflow — declaring variables and using them in a reusable VPC template.

### File Structure

```
project/
├── main.tf
└── variables.tf
```

---

### `main.tf`

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = var.vpc_name
  }
}
```

---

### `variables.tf`

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "vpc_name" {
  description = "Name tag for the VPC"
  default     = "cloudfolks-vpc"
}
```

---

### Run It

```bash
terraform init
terraform apply
```

### What Happens

- You did **not** provide any values manually
- Terraform saw the `default` values in `variables.tf`
- It used `vpc_cidr = 10.0.0.0/16` and `vpc_name = cloudfolks-vpc` automatically
- A VPC gets created with those values

---

## 7. The Standard File Convention

| File | Purpose |
|---|---|
| `main.tf` | The actual infrastructure logic — resources, providers |
| `variables.tf` | All variable declarations (name, type, default, description) |
| `terraform.tfvars` | (Optional) Actual values to pass to variables |

Keeping them separate makes your code **organized and readable**.

---

## 8. Key Takeaways

- Variables make Terraform code **reusable, clean, and easy to maintain**
- Always declare variables in `variables.tf` before using them
- Use the `var.` prefix to reference a variable anywhere in `.tf` files
- A variable with a `default` is **optional** — one without is **required**
- Variable properties like `type`, `description`, `sensitive`, and `validation` make your code safer and more descriptive

---

## Practice Exercises

### Exercise 1 — Basic Variable Declaration
Create a `variables.tf` file with two variables:
- `instance_type` with a default of `"t2.micro"`
- `region` with a default of `"us-east-1"`

Then create a `main.tf` that uses `var.region` in the AWS provider block.

---

### Exercise 2 — Required Variable
Declare a variable `environment` with **no default value** and use it as a tag in an `aws_vpc` resource. Run `terraform plan` and observe what Terraform does when no value is provided.

---

### Exercise 3 — Multiple Environments
You have one `main.tf` that creates an S3 bucket using `var.bucket_name`. Write the `variables.tf` file such that:
- For Dev: bucket name defaults to `"dev-my-bucket"`
- Think about how you would change it for QA or Prod without editing `main.tf`

---

### Exercise 4 — Sensitive Variable
Declare a variable called `db_password` with:
- `type = string`
- `sensitive = true`
- No default value

Write a short explanation of why you would mark it as `sensitive`.

---

### Exercise 5 — Variable with Description and Type
Declare the following variables with correct types and descriptions:
- `vpc_cidr` → type `string`, description: "CIDR block for VPC"
- `enable_dns` → type `bool`, default: `true`
- `subnet_count` → type `number`, default: `2`

Use all three in a resource block (you can write a mock block — it doesn't need to be a real AWS resource).

---

### Exercise 6 — Spot the Bug
What is wrong with the following code?

```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = vpc_cidr
  instance_type = variable.instance_type
}
```

Identify **two mistakes** and write the corrected version.

---

*Notes prepared from: Milestone 5 — Terraform Variables*
