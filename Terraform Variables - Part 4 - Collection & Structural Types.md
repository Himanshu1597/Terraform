# Terraform Variables — Part 4: Collection & Structural Types

> **Recap:** Part 3 covered Primitive Types (`string`, `number`, `bool`) — single values.  
> This file covers types that hold **multiple values** in one variable.

---

## Collection Types

Collection types let you store **multiple values of the same type** under a single variable name.

| Type | What It Stores | Ordered? | Duplicates? | Access By |
|---|---|---|---|---|
| `list` | Ordered values | Yes | Allowed | Index (`[0]`, `[1]`) |
| `set` | Unique values | No | Not allowed | Iteration only |
| `map` | Key-value pairs | No | Keys must be unique | Key name (`["key"]`) |

---

### `list` — Ordered Collection

A list holds values **in order**. You access them by their position, starting at index `0`.

**Common uses:** Availability Zones, CIDR blocks, Subnet IDs, Security Group IDs.

**Declaration (`variables.tf`):**
```hcl
variable "az_list" {
  description = "Two AZs in the chosen region"
  type        = list(string)
}
```

**Value (`terraform.tfvars`):**
```hcl
az_list = ["ap-south-1a", "ap-south-1b"]
```

**Usage (`main.tf`):**
```hcl
resource "aws_subnet" "subnet_az1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "192.168.0.0/25"
  availability_zone = var.az_list[0]   # "ap-south-1a"
}

resource "aws_subnet" "subnet_az2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "192.168.0.128/25"
  availability_zone = var.az_list[1]   # "ap-south-1b"
}
```

> `var.az_list[0]` gives the first item, `var.az_list[1]` gives the second. Index always starts at `0`.

---

### `set` — Unique Unordered Collection

A set holds values like a list, but **automatically removes duplicates** and has **no guaranteed order**.

**Common uses:** Unique tags, unique role names, allowed IPs — anywhere duplicates are meaningless.

**Declaration:**
```hcl
variable "allowed_roles" {
  type = set(string)
}
```

**Value:**
```hcl
allowed_roles = ["admin", "user", "admin"]
# Terraform stores this as: {"admin", "user"} — duplicate removed
```

**Important limitation:** Because sets have no order, you **cannot access items by index** (`var.my_set[0]` will error). Sets are designed for iteration with `for_each`, not direct indexing.

> The practical example of set with `for_each` is covered when loops are introduced. For now, understand what a set is and when to reach for it.

---

### `map` — Key-Value Pairs

A map stores values as **named pairs** — you look up a value by its key. All values in a map must be the same type.

**Common uses:** Environment-to-instance-type mappings, resource tags, region-to-AMI lookups.

**Declaration (`variables.tf`):**
```hcl
variable "env_instance_type" {
  type        = map(string)
  description = "Map of environment to EC2 instance type"
}

variable "environment" {
  type        = string
  description = "Environment to deploy (e.g., dev, prod)"
}
```

**Value (`terraform.tfvars`):**
```hcl
env_instance_type = {
  dev  = "t2.micro"
  prod = "t3.medium"
}

environment = "dev"
```

**Usage (`main.tf`):**
```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.env_instance_type[var.environment]   # picks "t2.micro" when env = "dev"
  tags = {
    Name = "Terraform-Map-Example"
    Env  = var.environment
  }
}
```

**How the lookup works:**
- `environment = "dev"` → `var.env_instance_type["dev"]` → `"t2.micro"`
- Change `environment = "prod"` → Terraform picks `"t3.medium"` automatically

---

## Structural Types

Structural types let you **group multiple values of different types** under one variable — like a form or a config block.

Both `tuple` and `object` serve this purpose. The difference is in how you define, provide, and access the values.

---

### `tuple` — Fixed-Length, Mixed-Type List

A tuple is like a list, but:
- **Fixed number of items** — you can't add or remove
- **Each position has its own type** — position 0 can be a string, position 1 a number, etc.
- **Access by index**, just like a list

**list vs tuple — key differences:**

| | `list` | `tuple` |
|---|---|---|
| Item types | All same type | Each position can differ |
| Length | Flexible | Fixed |
| Access | By index | By index |
| Type check | Checks item type | Checks order + type per position |
| Example | `["t2.micro", "t2.small"]` | `["t2.micro", 2, true]` |

**Declaration (`variables.tf`):**
```hcl
variable "server_info" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 2, true]
}
```

**Usage (`main.tf`):**
```hcl
resource "aws_instance" "demo_tuple" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.server_info[0]   # string → "t2.micro"
  count         = var.server_info[1]   # number → 2
  monitoring    = var.server_info[2]   # bool   → true
}
```

**Be careful:**
- Order matters — swapping positions breaks the logic
- You cannot skip or add extra values
- Hard to read at a glance: `var.server_info[0]` tells you nothing about what it means

---

### `object` — Named Fields with Mixed Types

An object is like a map, but:
- **Fixed, named keys** — field names are defined upfront
- **Each field can have its own type** — unlike map where all values must match
- **Access by field name** using dot notation

**map vs object — key differences:**

| | `map` | `object` |
|---|---|---|
| Keys | Any key, defined in values | Fixed, declared in type |
| Value types | All same type | Each field can differ |
| Access | `["key"]` | `.field_name` |
| Validation | Checks value types | Checks key names + value types |
| Best for | Tags, labels, uniform settings | Complex configs (EC2, VPC, S3) |

**Declaration (`variables.tf`):**
```hcl
variable "ec2_config" {
  type = object({
    instance_type = string
    count         = number
    monitoring    = bool
  })
  default = {
    instance_type = "t2.micro"
    count         = 2
    monitoring    = true
  }
}
```

**Usage (`main.tf`):**
```hcl
resource "aws_instance" "demo" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.ec2_config.instance_type   # dot notation
  count         = var.ec2_config.count
  monitoring    = var.ec2_config.monitoring
}
```

> Field names are self-documenting — `var.ec2_config.instance_type` is immediately clear, unlike `var.server_info[0]`.

---

### `tuple` vs `object` — When to Use Which

| | `tuple` | `object` |
|---|---|---|
| Structure | Positional (like a list) | Named fields (like a struct) |
| Access | `[0]`, `[1]`, `[2]` | `.instance_type`, `.count` |
| Readability | Hard to understand later | Easy to read and maintain |
| Validation | Position + type must match | Field name + type must match |
| Best for | Short, simple, fixed-position data | Complex configs with descriptive field names |

**Rule of thumb:** If you need to store more than 2–3 values together, prefer `object` over `tuple` — field names make the code readable for everyone.

---

## Choosing the Right Type — Quick Guide

```
Single value?
  → string / number / bool  (Primitive)

Multiple values, all same type, need ordering and index access?
  → list(string), list(number), etc.

Multiple values, all same type, need uniqueness, will iterate with for_each?
  → set(string), etc.

Key-value pairs, all values same type (e.g., tags)?
  → map(string)

Fixed set of values in a specific order, different types, short?
  → tuple([string, number, bool])

Named fields, different types, descriptive, complex config?
  → object({ field = type, ... })
```

---

## Practice Exercises

### Exercise 1 — List Indexing
Declare a variable `subnet_cidrs` of type `list(string)` with three CIDR values. Write the `main.tf` code that creates three subnets using `var.subnet_cidrs[0]`, `[1]`, and `[2]`. What happens if you try `var.subnet_cidrs[3]`?

---

### Exercise 2 — Map Lookup
You have this variable:
```hcl
variable "region_ami" {
  type = map(string)
  default = {
    "us-east-1" = "ami-111"
    "us-west-2" = "ami-222"
    "ap-south-1" = "ami-333"
  }
}
```
Write a resource that picks the correct AMI based on a separate `variable "region"`. What happens if `region = "eu-west-1"` (a key that doesn't exist in the map)?

---

### Exercise 3 — set vs list
You receive this list from a user:
```hcl
["web", "db", "web", "cache", "db"]
```
If you store this as `list(string)`, what do you get? If you store it as `set(string)`, what do you get? Which would you use if you want to create one resource per unique value using `for_each`?

---

### Exercise 4 — Object Variable
You're building a reusable EC2 module. Instead of declaring separate variables for `instance_type`, `disk_size`, and `enable_monitoring`, group them into one `object` variable called `ec2_settings`. Write:
1. The `variables.tf` declaration
2. Sensible defaults
3. How you'd access each field in `main.tf`

---

### Exercise 5 — tuple vs object
A colleague stores EC2 config as:
```hcl
variable "config" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 1, false]
}
```
Six months later, a new team member asks: "What does `var.config[2]` mean?" Rewrite this as an `object` so the meaning is immediately clear without needing documentation.

---

*Notes prepared from: Milestone 5 — PDFs 9 & 10*  
*Type conversion functions (`tostring`, `tonumber`, `tobool`, `tolist`, `toset`, `tomap`) are covered in Part 5.*
