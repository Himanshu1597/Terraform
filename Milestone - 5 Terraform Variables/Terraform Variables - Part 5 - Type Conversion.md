# Terraform Variables — Part 5: Type Conversion

> Sometimes a value arrives in one type but a resource expects another. Terraform gives you 6 built-in functions to convert between types.

---

## Why Type Conversion Exists

When values come from `.tfvars`, environment variables, or external JSON, **they often arrive as strings by default** — even if the value looks like a number or boolean. Type conversion functions let you explicitly cast a value to the type a resource actually needs.

**6 conversion functions, split into two groups:**

| Group | Functions |
|---|---|
| Primitive | `tostring()`, `tonumber()`, `tobool()` |
| Collection | `tolist()`, `toset()`, `tomap()` |

---

## Primitive Conversions

---

### `tostring()` — Convert to String

Converts any value (number, bool) into text.

```hcl
tostring(101)    # → "101"
tostring(true)   # → "true"
```

**When to use:** When you need to embed a number inside a string expression — like building a dynamic bucket name.

```hcl
variable "student_id" {
  type    = number
  default = 101
}

resource "aws_s3_bucket" "demo" {
  bucket = "cloudfolks-${tostring(var.student_id)}-demo"
}
# bucket name → "cloudfolks-101-demo"
```

**Reality check:** In most day-to-day AWS resources, Terraform auto-converts numbers to strings inside string expressions. You will rarely need `tostring()` explicitly — it's mainly useful when combining values with `join()` or `format()`.

---

### `tonumber()` — Convert to Number

Converts a string into a numeric value.

```hcl
tonumber("2")    # → 2
tonumber("3.14") # → 3.14
```

**When to use:** When a value comes in as a string (from a `map(string)`, env variable, or external config) but a resource argument expects a number — like `count`, `volume_size`, or `port`.

```hcl
variable "ec2_config_map" {
  type = map(string)
  default = {
    instance_type = "t2.micro"
    count         = "2"        # stored as string
    environment   = "Dev"
  }
}

resource "aws_instance" "demo" {
  ami           = "ami-0861f4e788f5069dd"
  instance_type = var.ec2_config_map["instance_type"]
  count         = tonumber(var.ec2_config_map["count"])   # converted to number
}
```

> This pattern is common when all config comes from a `map(string)` — you pick the key and convert the value to the type you need.

---

### `tobool()` — Convert to Boolean

Converts a string `"true"` or `"false"` into an actual boolean `true` / `false`.

```hcl
tobool("true")   # → true
tobool("false")  # → false
```

**When to use:** When a boolean flag is stored or passed as a string — for example, from environment variables or a `map(string)` config.

```hcl
variable "enable_monitoring_str" {
  type    = string
  default = "true"
}

resource "aws_instance" "demo" {
  ami           = "ami-0861f4e788f5069dd"
  instance_type = "t2.micro"
  monitoring    = tobool(var.enable_monitoring_str)
}
```

> Note: `"true"` (string) and `true` (bool) are different. Without `tobool()`, passing a string to a `bool` argument throws a type error.

---

## Collection Conversions

---

### `tolist()` — Convert to List

Converts a value (typically a set) into a list so you can **access items by index**.

**Why this matters:** Sets have no guaranteed order, so Terraform does not allow indexing (`set[0]` errors). If you need to pick a specific item by position, convert to a list first.

```hcl
# Wrong — cannot index a set directly
variable "my_set" {
  type    = set(string)
  default = ["apple", "banana", "apple"]
}

output "wrong_way" {
  value = var.my_set[0]   # ERROR ❌
}
```

```hcl
# Right — convert set to list, then index
output "right_way" {
  value = tolist(var.my_set)[0]   # works ✅
}
# my_set after dedup → {"apple", "banana"}
# tolist() → ["apple", "banana"]
# [0] → "apple"
```

---

### `toset()` — Convert to Set

Converts a list (which may have duplicates) into a set, **automatically removing duplicates**.

```hcl
variable "my_list" {
  type    = list(string)
  default = ["a", "a", "b"]
}

output "original_list" {
  value = var.my_list          # → ["a", "a", "b"]
}

output "converted_set" {
  value = toset(var.my_list)   # → {"a", "b"}  (duplicate removed)
}
```

**Most common real-world use:** `for_each` in Terraform works natively with sets. If you have a list with possible duplicates and want to create one resource per unique value, convert to a set first.

```hcl
# for_each example (preview — covered in detail with loops)
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "alice"])   # "alice" created only once
  name     = each.value
}
```

---

### `tomap()` — Convert to Map

Converts structured values into a proper key-value map so Terraform can treat them as a `map` type.

```hcl
tomap({
  Name        = "my-instance"
  Environment = "dev"
})
```

**Practical coverage deferred:** To use `tomap()` meaningfully, you need to understand `for` loops and `split()` first. Hands-on examples will be covered when loops are introduced.

---

## Full Reference — All 6 Functions

| Function | Converts From → To | Common Use Case |
|---|---|---|
| `tostring(x)` | number / bool → string | Building dynamic resource names |
| `tonumber(x)` | string → number | Using string map values in `count`, `port`, `size` |
| `tobool(x)` | string → bool | Using string map values in `monitoring`, `encrypted` |
| `tolist(x)` | set → list | Indexing into a set (`[0]`, `[1]`) |
| `toset(x)` | list → set | Deduplicating before `for_each` |
| `tomap(x)` | object / other → map | Dynamic map creation (covered with loops) |

---

## Key Rules to Remember

- Terraform reads `.tfvars` and environment variable values as **strings by default**
- If a resource argument expects `number` or `bool`, and your variable is `map(string)`, you need `tonumber()` or `tobool()` at the point of use
- You cannot index a `set` directly — use `tolist()` first
- `tostring()` is rarely needed since Terraform auto-converts in string interpolations (`"${var.num}"`)
- `tomap()` and `toset()` are especially useful when working with `for_each`

---

## Practice Exercises

### Exercise 1 — `tonumber()` in practice
You have this map variable:
```hcl
variable "rds_config" {
  type = map(string)
  default = {
    allocated_storage = "20"
    backup_retention  = "7"
    port              = "5432"
  }
}
```
Write the `main.tf` code that creates an RDS instance using these values, applying `tonumber()` where required. Which fields need conversion and why?

---

### Exercise 2 — `tobool()` from environment variable
A CI/CD pipeline sets an environment variable `TF_VAR_skip_final_snapshot="true"`. The RDS resource argument `skip_final_snapshot` expects a `bool`. Declare the variable with `type = string` and write the resource argument using `tobool()`.

---

### Exercise 3 — `tolist()` vs direct set access
Given:
```hcl
variable "az_set" {
  type    = set(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```
1. Why does `var.az_set[0]` fail?
2. Write the correct way to get the first element.
3. Is the first element guaranteed to be `"us-east-1a"`? Why or why not?

---

### Exercise 4 — `toset()` for deduplication
A teammate passes this list for environment names:
```hcl
env_list = ["dev", "prod", "dev", "staging", "prod"]
```
Write an output block that shows the deduplicated set. How many unique environments are there?

---

### Exercise 5 — Pick the right function
For each scenario below, identify which conversion function (if any) is needed:

1. Variable is `type = number`, used in `bucket = "app-${var.version}-bucket"`
2. Variable is `map(string)`, key `"enable_encryption"` used in `encrypted = ?`
3. Variable is `set(string)`, need to pass it to a module that expects `list(string)`
4. Variable is `list(string)` with possible duplicates, used with `for_each`
5. Variable is `type = bool`, used in `monitoring = var.enable_monitoring`

---

*Notes prepared from: Milestone 5 — PDF 11*  
*This completes the Terraform Variables series (Parts 1–5).*
