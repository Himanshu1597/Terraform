# Terraform Variables — Part 7: Sensitive Variables & Secret Security

---

## What is a Sensitive Variable?

A sensitive variable is one whose value Terraform **deliberately hides** from:
- CLI output (`terraform plan`, `terraform apply`)
- Terraform logs
- The Terraform UI

Values like passwords, access keys, API tokens, and private keys should **never appear in plain text** in any terminal output or log file. The `sensitive = true` flag is Terraform's built-in mechanism to enforce this.

---

## Syntax

Add `sensitive = true` inside the variable block:

```hcl
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}
```

That's all. Terraform will now treat the value as secret and hide it wherever it appears.

The same flag works on **output blocks** too:

```hcl
output "rds_password" {
  value     = var.db_password
  sensitive = true
}
```

---

## The Three Combinations — Behavior Table

| Variable | Output | Input Behavior | Output Behavior |
|---|---|---|---|
| `sensitive = true` | `sensitive = true` | Hidden when typing | Shows `(sensitive value)` ✅ |
| `sensitive = true` | No `sensitive` flag | Hidden when typing | Terraform **errors** ❌ |
| No `sensitive` flag | `sensitive = true` | Visible when typing | Shows `(sensitive value)` ⚠️ |

---

### Combination 1 — Sensitive Variable + Sensitive Output (Safe)

```hcl
# variables.tf
variable "secret_var" {
  type      = string
  sensitive = true
}

# outputs.tf
output "secret_output" {
  value     = var.secret_var
  sensitive = true
}
```

**Result:**
- Input is hidden when you type it
- Output shows `(sensitive value)` — value never exposed

---

### Combination 2 — Sensitive Variable + Normal Output (Error)

```hcl
# variables.tf
variable "secret_var" {
  type      = string
  sensitive = true
}

# outputs.tf
output "secret_output" {
  value = var.secret_var   # no sensitive = true
}
```

**Result:**
```
Error: Output refers to sensitive values
```

Terraform refuses to apply. It will not let you accidentally expose a sensitive variable through an unsecured output.

---

### Combination 3 — Normal Variable + Sensitive Output (Half Protected)

```hcl
# variables.tf
variable "normal_var" {
  type = string   # no sensitive = true
}

# outputs.tf
output "hidden_output" {
  value     = var.normal_var
  sensitive = true
}
```

**Result:**
- Input is **visible** while typing (not protected at input)
- Output shows `(sensitive value)` (protected at output)

This gives partial protection — better than nothing, but the value could already have been seen at input time.

---

## Important: `sensitive = true` Does NOT Encrypt the State File

This is a common misunderstanding.

`sensitive = true` only **hides the value from CLI output**. The value is **still stored in plain text** inside `terraform.tfstate`. Anyone who can read the state file can see the secret.

This is why securing the state file (covered below) is equally important.

---

## When to Use `sensitive = true` in AWS

| Scenario | Example |
|---|---|
| **Database passwords** | RDS, Aurora, Redshift username/password |
| **AWS access & secret keys** | IAM user credentials for programmatic access |
| **API keys / tokens** | Stripe, Twilio, GitHub tokens used by Lambda/EC2/ECS |
| **Encryption keys & secrets** | KMS keys, S3 encryption keys, Secrets Manager values |

---

## Best Practices for Secret Security

---

### Best Practice 1 — Never Hardcode Secrets

```hcl
# BAD — never do this
resource "aws_db_instance" "example" {
  username = "admin"
  password = "Hardcoded@123"   # ❌ visible to anyone with repo access
}
```

**Why it's dangerous:**
- Stored in plain text in source code
- Anyone with repo access sees it immediately
- Leaks into version control history — even if you delete the line later, it remains in git history

**What to do instead:** Use a variable, and provide the value through a `.tfvars` file, environment variable, or a secrets manager — never inline in `.tf` files.

---

### Best Practice 2 — Mark Secrets as `sensitive = true`

Even after moving secrets to `.tfvars`, Terraform can still show them in `plan`, `apply`, and output. Always mark secret variables and their outputs with `sensitive = true`.

```hcl
# variables.tf
variable "db_password" {
  type      = string
  sensitive = true
}

# outputs.tf
output "rds_password" {
  value     = var.db_password
  sensitive = true
}
```

CLI will show:
```
Outputs:
rds_password = <sensitive>
```

---

### Best Practice 3 — Use a Secrets Manager (Strongest Option)

Even a `.tfvars` file is a file on disk — it can be accidentally committed, copied, or leaked. The strongest approach is to **pull secrets dynamically at runtime** from a dedicated secrets management tool.

| Tool | What It Does |
|---|---|
| **AWS SSM Parameter Store** | Store encrypted secrets as SecureString parameters |
| **AWS Secrets Manager** | Manage secrets with automatic rotation |
| **HashiCorp Vault** | Full-featured enterprise secret manager |
| **GitHub / GitLab Secrets** | CI/CD built-in secure vaults for pipeline secrets |

**Example — AWS SSM Parameter Store:**

```hcl
# Pull secret from SSM at runtime — nothing stored in .tf or .tfvars
data "aws_ssm_parameter" "db_pass" {
  name            = "/prod/rds/password"
  with_decryption = true
}

resource "aws_db_instance" "example" {
  password = data.aws_ssm_parameter.db_pass.value
}
```

**Advantages:**
- No secret stored in any `.tf` or `.tfvars` file
- Secret is pulled securely at runtime
- Centralized management — rotate once, all consumers get the new value

---

### Best Practice 4 — Secure the Terraform State File

`terraform.tfstate` stores **all resource values including secrets in plain text**, regardless of `sensitive = true`. This makes it one of the most sensitive files in your project.

**Never do:**
- Store `.tfstate` on a local laptop
- Commit `.tfstate` to Git or any version control

**What to do instead:**

| Action | Why |
|---|---|
| **Use a remote backend** (e.g., S3) | Centralized, access-controlled, not on anyone's machine |
| **Enable encryption** (S3 + KMS) | Secrets encrypted at rest |
| **Enable state locking** (DynamoDB) | Prevents two people running apply at the same time |
| **Set IAM permissions** | Only authorized people/pipelines can read the state |

---

### Best Practice 5 — Don't Output Secrets Unless Necessary

Even with `sensitive = true`, outputting a secret increases risk — it can still appear in automation pipelines, CI/CD logs, or be accessed by anyone with permissions to read Terraform outputs.

```hcl
# Dangerous — exposes secret in output
output "db_password" {
  value     = var.db_password
  sensitive = false   # ❌
}

# Safer — hides from CLI, but still ask: do you need this output at all?
output "db_password" {
  value     = var.db_password
  sensitive = true   # ✅
}
```

Only output a secret when:
- It's absolutely required (e.g., passing the value to another system)
- You're debugging in a controlled, safe environment
- The workspace and team access are properly locked down

If you don't need the output, don't write it.

---

## Passing Secrets Without Files — Environment Variables

Another safe way to inject secrets is via `TF_VAR_` environment variables. Nothing is written to any file.

```bash
export TF_VAR_db_password="MySecurePass123"
terraform apply
```

Terraform automatically picks up `TF_VAR_db_password` and maps it to `var.db_password`. Useful in CI/CD pipelines where secrets are injected via the pipeline's secret store (GitHub Secrets, GitLab CI Variables, etc.).

---

## `.gitignore` — Must-Have for Any Terraform Project

Always add these to your `.gitignore` to prevent accidental commits:

```
# .gitignore
*.tfvars
*.tfvars.json
*.tfstate
*.tfstate.backup
.terraform/
```

> `*.tfvars` excludes secret values from being committed. `*.tfstate` keeps the state file out of version control. `.terraform/` excludes the downloaded provider binaries.

---

## Summary — Secrets Security Checklist

| Rule | Why |
|---|---|
| Never hardcode secrets in `.tf` files | Plain text, visible in Git history |
| Always mark sensitive variables with `sensitive = true` | Hides from CLI, plan, and apply output |
| Always pair with `sensitive = true` on outputs | Terraform will error if you don't |
| Don't rely on `sensitive = true` for state security | It doesn't encrypt the state file |
| Use remote backend with encryption for state | Protects secrets stored in state |
| Prefer secrets managers over `.tfvars` files | Dynamic, centralized, no file on disk |
| Use `TF_VAR_` env vars in CI/CD pipelines | No secret files needed in pipelines |
| Add `*.tfvars` and `*.tfstate` to `.gitignore` | Prevents accidental commits |

---

## Practice Exercises

### Exercise 1 — Mark and Use a Sensitive Variable
Declare a variable `api_token` with `type = string` and `sensitive = true`. Use it in a resource tag. Write the output block that safely exposes it. What happens if you forget `sensitive = true` on the output?

---

### Exercise 2 — Spot the Security Mistake
What is wrong with this configuration?

```hcl
resource "aws_db_instance" "main" {
  engine         = "mysql"
  instance_class = "db.t3.micro"
  username       = "root"
  password       = "SuperSecret99!"
}
```

Rewrite it correctly using a variable, `terraform.tfvars`, and `sensitive = true`.

---

### Exercise 3 — State File Risk
A developer stores `terraform.tfstate` in a public GitHub repo "because the infra is not critical yet." The state contains an RDS password marked `sensitive = true` in `variables.tf`. Is the password safe? Explain why or why not.

---

### Exercise 4 — SSM vs tfvars
You're setting up a production RDS instance. Compare two approaches:
- Storing the DB password in `secrets.tfvars` with `sensitive = true`
- Pulling it from AWS SSM Parameter Store using a `data` block

For each: what are the risks, and which is more appropriate for production?

---

### Exercise 5 — CI/CD Secret Injection
Your team uses GitHub Actions to run `terraform apply`. You need to pass `db_password` without storing it in any file. Write the GitHub Actions step that sets the environment variable correctly, and the matching Terraform variable declaration.

---

*Notes prepared from: Milestone 5 — PDFs 13 & 14*  
*This completes the Terraform Variables series (Parts 1–7).*
