# Terraform Variables — Part 6: Custom Variable Validation

---

## Why Custom Validation?

Declaring `type = number` already ensures only numbers are accepted. But Terraform will still accept **any** number — 1, 99, 1000, -5.

If you need to restrict **which values are valid** (e.g. only `2` or `4`, only `"dev"/"prod"`, only values between 1–100), you need a **custom validation block**.

---

## Syntax

A `validation` block sits inside the `variable` block. It has exactly two fields:

```hcl
variable "variable_name" {
  type        = <type>
  description = "<description>"

  validation {
    condition     = <boolean expression using var.variable_name>
    error_message = "Message shown to the user when validation fails."
  }
}
```

| Field | What It Does |
|---|---|
| `condition` | An expression that must evaluate to `true` for the value to be accepted |
| `error_message` | The message Terraform shows if `condition` is `false` |

**Rules:**
- `condition` must return `true` (valid) or `false` (invalid)
- `condition` can only reference the variable being validated — not other variables
- `error_message` must be a non-empty string
- You can have **multiple** `validation` blocks on one variable (each checked independently)
- Terraform checks validation **before** creating any resources — it fails fast at plan/apply time

---

## Condition Operators Quick Reference

| Operator | Meaning | Example |
|---|---|---|
| `==` | Equal to | `var.x == 2` |
| `!=` | Not equal to | `var.x != 0` |
| `>`, `>=` | Greater than / or equal | `var.x >= 1` |
| `<`, `<=` | Less than / or equal | `var.x <= 100` |
| `&&` | AND — both must be true | `var.x >= 1 && var.x <= 10` |
| `\|\|` | OR — at least one must be true | `var.x == 2 \|\| var.x == 4` |
| `!` | NOT — flips true/false | `!contains([...], var.x)` |
| `%` | Modulo (remainder) | `var.x % 2 == 0` means even |

---

## Number Validation Examples

### Allow only specific values — `==` with `||`

```hcl
variable "instance_count" {
  type        = number
  description = "Number of EC2 instances. Only 2 or 4 allowed."

  validation {
    condition     = var.instance_count == 2 || var.instance_count == 4
    error_message = "Only 2 or 4 are valid values for instance_count."
  }
}
```

- `var.instance_count == 2` → is the value 2?
- `||` → OR
- `var.instance_count == 4` → is the value 4?
- Accepts: `2`, `4` — Rejects: `1`, `3`, `10`

---

### Allow a range — `&&` with `>=` and `<=`

```hcl
validation {
  condition     = var.instance_count >= 1 && var.instance_count <= 100
  error_message = "instance_count must be between 1 and 100."
}
```

- `>= 1` → value must be 1 or more
- `&&` → AND (both conditions must be true)
- `<= 100` → value must be 100 or less
- Accepts: `1`, `50`, `100` — Rejects: `0`, `101`

---

### Allow values from a fixed list — `contains()`

```hcl
validation {
  condition     = contains([2, 4, 8], var.instance_count)
  error_message = "instance_count must be 2, 4, or 8."
}
```

`contains(list, value)` — returns `true` if the value exists in the list.

- Accepts: `2`, `4`, `8` — Rejects: `1`, `3`, `16`
- Cleaner than chaining multiple `==` conditions with `||`

---

### Allow only even numbers in a range — `%` modulo

```hcl
validation {
  condition     = var.instance_count % 2 == 0 && var.instance_count >= 1 && var.instance_count <= 10
  error_message = "instance_count must be an even number between 1 and 10."
}
```

- `% 2 == 0` → remainder after dividing by 2 is 0 → the number is even
  - `2 % 2 = 0` → even ✅
  - `3 % 2 = 1` → odd ❌
- Combined with range: value must be even AND between 1–10
- Accepts: `2`, `4`, `6`, `8`, `10` — Rejects: `1`, `3`, `9`, `12`

---

### Exclude specific values — `!contains()`

```hcl
validation {
  condition     = !contains([13, 99], var.instance_count)
  error_message = "instance_count cannot be 13 or 99."
}
```

- `contains([13, 99], var.instance_count)` → true if value IS 13 or 99
- `!` → flips it — condition passes only when value is NOT 13 or 99
- Useful for blacklisting specific values

---

## String Validation Examples

### Allow only specific strings — `contains()`

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "Environment must be dev, test, or prod."
  }
}
```

- Accepts: `"dev"`, `"test"`, `"prod"`
- Rejects: `"qa"`, `"prod1"`, `"production"`, `"DEV"` (case-sensitive)

---

### Minimum length — `length()`

```hcl
variable "name" {
  type = string

  validation {
    condition     = length(var.name) >= 3
    error_message = "Name must be at least 3 characters long."
  }
}
```

`length(string)` returns the number of characters.

- Accepts: `"abc"`, `"hello"`
- Rejects: `"a"`, `"hi"`

---

### Prefix check — `startswith()`

```hcl
variable "env_name" {
  type = string

  validation {
    condition     = startswith(var.env_name, "dev")
    error_message = "Environment name must start with 'dev'."
  }
}
```

- Accepts: `"dev-app"`, `"dev01"`, `"development"`
- Rejects: `"test-dev"`, `"production"`

> **Note:** The PDF uses `substr(var.env_name, 0, 3) == "dev"` which also works.  
> `substr(string, offset, length)` — offset `0`, length `3` returns the first 3 characters.  
> `startswith()` is cleaner and available in Terraform 1.3+.

---

### Pattern matching — `can(regex())`

Not in the PDF, but worth knowing for real projects:

```hcl
variable "ami_id" {
  type = string

  validation {
    condition     = can(regex("^ami-[0-9a-f]{8,17}$", var.ami_id))
    error_message = "ami_id must be a valid AMI ID like ami-0c55b159cbfafe1f0."
  }
}
```

- `regex(pattern, string)` — matches a pattern; errors if no match
- `can(expression)` — returns `true` if the expression succeeds, `false` if it errors
- Together: `can(regex(...))` gives a safe boolean for use in `condition`

---

## Multiple Validation Blocks

You can stack multiple `validation` blocks on one variable. Each is checked independently.

```hcl
variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1
    error_message = "instance_count must be at least 1."
  }

  validation {
    condition     = var.instance_count <= 10
    error_message = "instance_count must not exceed 10."
  }

  validation {
    condition     = var.instance_count % 2 == 0
    error_message = "instance_count must be an even number."
  }
}
```

Each block reports its own `error_message` independently, which helps the user know exactly which rule failed.

---

## What Happens When Validation Fails

If the `condition` is `false`, Terraform stops immediately and shows:

```
╷
│ Error: Invalid value for variable
│
│   on variables.tf line 1:
│    1: variable "instance_count" {
│
│ Only 2 or 4 are valid values for instance_count.
│
│ This was checked by the validation rule at variables.tf:6,3-13.
╵
```

- This happens **before** any resource is created or modified
- You fix the value in `terraform.tfvars` (or wherever you're passing it) and re-run

---

## Complete Working Example

`variables.tf`
```hcl
variable "instance_count" {
  type        = number
  description = "Number of EC2 instances. Only 2 or 4 allowed."

  validation {
    condition     = contains([2, 4], var.instance_count)
    error_message = "Only 2 or 4 are valid values for instance_count."
  }
}
```

`main.tf`
```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_instance" "example" {
  count         = var.instance_count
  ami           = "ami-0861f4e788f5069dd"
  instance_type = "t2.micro"
  tags = {
    Name = "ValidatedEC2"
  }
}
```

`terraform.tfvars`
```hcl
instance_count = 3   # ❌ fails validation — must be 2 or 4
```

Change to:
```hcl
instance_count = 2   # ✅ passes
```

---

## When to Use Custom Validation

| Scenario | Validation to Use |
|---|---|
| Allow only specific values | `contains([...], var.x)` |
| Enforce a range | `var.x >= min && var.x <= max` |
| Allow only even/odd numbers | `var.x % 2 == 0` |
| Blacklist certain values | `!contains([...], var.x)` |
| Enforce string prefix | `startswith(var.x, "prefix")` |
| Enforce minimum length | `length(var.x) >= n` |
| Validate format (AMI ID, CIDR) | `can(regex("pattern", var.x))` |

---

## Practice Exercises

### Exercise 1 — Allowed Environments
Declare a variable `environment` of type `string`. Write a validation that only allows `"dev"`, `"staging"`, or `"prod"`. What error does Terraform show if someone passes `"production"`?

---

### Exercise 2 — Port Range
Declare a variable `app_port` of type `number`. Write a validation that ensures the port is between `1024` and `65535` (valid non-privileged port range).

---

### Exercise 3 — Even Instance Count
Declare `instance_count` as a `number`. Write a validation that:
- Allows only even numbers
- Allows only values from 2 to 20

Write it as a single `condition` using `&&`.

---

### Exercise 4 — Prefix Enforcement
Your team has a naming convention: all S3 bucket name variables must start with `"cf-"`. Write the variable declaration with the correct validation using `startswith()`. What values would pass and what would fail?

---

### Exercise 5 — Multiple Validation Blocks
Declare a variable `username` of type `string` with three separate validation blocks:
1. Must be at least 4 characters
2. Must be no more than 20 characters
3. Must start with `"user-"`

Write all three blocks and give examples of values that pass all three and values that fail each one individually.

---

*Notes prepared from: Milestone 5 — PDF 12*
