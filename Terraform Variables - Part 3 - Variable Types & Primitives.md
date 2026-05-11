# Terraform Variables — Part 3: Variable Types & Primitive Types

---

## 1. Why Define a Type for a Variable?

When you declare a variable, you can (and should) tell Terraform **what kind of data** that variable holds — a piece of text, a number, a true/false flag, etc.

By defining the type, Terraform:
- **Validates the input** — it checks that the value you provide matches the expected type
- **Catches mistakes early** — if someone passes a string where a number is expected, Terraform throws an error before anything gets created
- **Improves readability** — anyone reading the code immediately knows what kind of value is expected

### Is the Type Required?

No. The `type` field is **optional**. If you omit it, Terraform tries to infer the type from the default value or the input provided. But it's always a good habit to define it explicitly — it prevents silent bugs and makes the code self-documenting.

> **Rule:** If you declare `type = bool`, your default value must be `true` or `false`. Default values must always match the declared type.

---

## 2. The Three Categories of Variable Types

Terraform organizes variable types into three groups:

| Category | What It Stores | Types |
|---|---|---|
| **Primitive** | A single simple value | `string`, `number`, `bool` |
| **Collection** | Multiple values of the same type | `list`, `set`, `map` |
| **Structural** | Complex, mixed-type data | `object`, `tuple` |

This file covers **Primitive Types**. Collection and Structural types are covered in Part 4.

---

## 3. Primitive Types

Primitive types store **one value at a time** — a single piece of text, a single number, or a single true/false. They are the most commonly used variable types in Terraform.

---

### `string` — Text Values

A `string` is any text wrapped in double quotes.

**Common uses:** region names, instance types, AMI IDs, resource names, environment labels.

**Declaration (`variables.tf`):**
```hcl
variable "instance_type" {
  type = string
}
```

**Usage (`main.tf`):**
```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
}
```

**Value (`terraform.tfvars`):**
```hcl
instance_type = "t2.micro"
```

> Strings must always be in double quotes — both in the variable value and in the `.tfvars` file.

---

### `number` — Numeric Values

A `number` can be any integer or float (decimal).

**Common uses:** instance count, port numbers, disk size, timeout values.

**Declaration (`variables.tf`):**
```hcl
variable "instance_count" {
  type = number
}
```

**Usage (`main.tf`):**
```hcl
resource "aws_instance" "example" {
  count         = var.instance_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**Value (`terraform.tfvars`):**
```hcl
instance_count = 2
```

> Numbers are written **without quotes** in `.tfvars`. Writing `instance_count = "2"` would cause a type mismatch error.

---

### `bool` — True / False Values

A `bool` (boolean) holds only one of two values: `true` or `false`.

**Common uses:** feature flags, enabling/disabling monitoring, toggling encryption, public access settings.

**Declaration (`variables.tf`):**
```hcl
variable "enable_monitoring" {
  type = bool
}
```

**Usage (`main.tf`):**
```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  monitoring    = var.enable_monitoring
}
```

**Value (`terraform.tfvars`):**
```hcl
enable_monitoring = true
```

> Booleans are written **without quotes** — `true` not `"true"`. Quoting them turns them into strings.

---

## 4. Primitive Types — Side-by-Side Summary

| Type | Stores | Example Value | Quotes in `.tfvars`? |
|---|---|---|---|
| `string` | Text | `"t2.micro"` | Yes |
| `number` | Integer or float | `2`, `3.14` | No |
| `bool` | True or false | `true`, `false` | No |

---

## 5. The `any` Type — Worth Knowing

Terraform has a special type called `any`. When you use it, Terraform accepts any kind of value and determines the type at runtime.

```hcl
variable "flexible_value" {
  type = any
}
```

This is flexible but loses the validation benefit. Avoid using `any` unless you have a specific reason — explicit types make code safer and clearer.

---

## 6. Complete Example with All Three Primitive Types

`variables.tf`
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}
```

`terraform.tfvars`
```hcl
instance_type     = "t3.medium"
instance_count    = 3
enable_monitoring = true
```

`main.tf`
```hcl
resource "aws_instance" "example" {
  count         = var.instance_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  monitoring    = var.enable_monitoring
}
```

---

## 7. Key Rules — Quick Reference

- Always define `type` explicitly — it's optional but strongly recommended
- String values need **double quotes**; numbers and bools do not
- Default values **must match** the declared type
- If you skip `type`, Terraform infers it — but this can lead to silent type mismatches
- Use `any` sparingly — it bypasses type validation

---

## Practice Exercises

### Exercise 1 — Declare All Three Primitive Types
Write a `variables.tf` file with these three variables:
- `region` — type `string`, default `"us-east-1"`
- `disk_size_gb` — type `number`, default `20`
- `enable_public_ip` — type `bool`, default `true`

Then write a matching `terraform.tfvars` that overrides all three defaults.

---

### Exercise 2 — Spot the Type Mismatch
What is wrong with each of these?

```hcl
# variables.tf
variable "port" {
  type    = number
  default = "8080"
}

variable "is_private" {
  type    = bool
  default = "false"
}
```

Identify the issue in each and write the corrected version.

---

### Exercise 3 — Use Type Validation in Practice
Declare a variable `environment` with `type = string` and no default. What does Terraform do when you:
1. Run `terraform plan` without providing a value?
2. Run `terraform plan -var="environment=prod"`?
3. Run `terraform plan -var="environment=42"`?

Predict the outcome for each case and explain why.

---

### Exercise 4 — Real-World Variable Setup
You're provisioning an EC2 instance and need these inputs:
- The AMI ID (text)
- How many instances to create (whole number)
- Whether to enable termination protection (true/false)

Write the `variables.tf` with proper types, descriptions, and sensible defaults. Then write the `terraform.tfvars` for a production environment where you want 2 instances with termination protection enabled.

---

### Exercise 5 — `any` vs Explicit Types
A colleague wrote this:

```hcl
variable "port" {
  type    = any
  default = 80
}
```

Then in `terraform.tfvars` someone accidentally sets:
```hcl
port = "eighty"
```

What could go wrong? Rewrite the variable declaration to catch this mistake at the variable level, before Terraform tries to use it in a resource.

---

*Notes prepared from: Milestone 5 — PDFs 7 & 8*
*Collection Types (list, set, map) and Structural Types (object, tuple) are covered in Part 4.*
