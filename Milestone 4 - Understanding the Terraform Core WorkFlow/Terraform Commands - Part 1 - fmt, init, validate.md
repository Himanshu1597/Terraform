# Terraform Commands — Part 1: `fmt`, `init`, `validate`

> The three commands you run **before** deploying anything. Format your code, initialize the project, and validate it works — all without touching AWS.

---

## The Standard Pre-Deployment Workflow

```
1. terraform fmt        →  clean up formatting
2. terraform init       →  download providers, set up project
3. terraform validate   →  check code for syntax/logic errors
   (then: terraform plan → apply)
```

---

## 1. `terraform fmt` — Format the Code

### What It Does

Auto-formats all `.tf` files in the current directory to follow Terraform's official style — like a code beautifier. It fixes:
- **Indentation** — aligns blocks, brackets, spacing
- **Style** — applies Terraform's official formatting conventions
- **Alignment** — lines up `=` signs in argument lists

### When to Run It

| Situation | Run It? |
|---|---|
| After writing `.tf` files | ✅ Yes |
| Before `terraform init` | ✅ Yes |
| Before sharing code with the team | ✅ Yes |
| After every small edit | Not required, but a good habit |

### How to Use It

```bash
terraform fmt              # format files in the current folder
terraform fmt -recursive   # also format files in subfolders
terraform fmt -check       # check if formatting is needed (no changes made) — useful for CI
terraform fmt -diff        # show what would change
```

> If no changes are needed, `terraform fmt` produces **no output** — that means everything is already clean.

### Before vs After Example

**Before:**
```hcl
provider "aws" {
region = "ap-south-1"
}
resource "aws_instance" "web" {
ami = "ami-0f918f7e67a3323f0"
instance_type = "t2.micro"
tags = {
Name = "MyFirstEC2"
}
}
```

**After:**
```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0f918f7e67a3323f0"
  instance_type = "t2.micro"
  tags = {
    Name = "MyFirstEC2"
  }
}
```

### When `fmt` Rewrites the File

It rewrites in place when:
- The file is valid HCL syntax
- The only issues are formatting (spacing, indentation, alignment)

### When `fmt` Throws Errors

`fmt` will fail if the file is structurally broken:
- Mismatched braces `{ }`
- Missing equal signs `=`
- Missing newline after an argument
- Strings missing quotes

**Example that breaks:**
```hcl
provider "aws" {
  region = "ap-south-1"}
```
```
Error: Missing newline after argument
An argument definition must end with a newline
```

### Important — What `fmt` Does NOT Do

A passing `terraform fmt` only proves the file is **formatted correctly and is valid HCL syntax**. It does NOT catch:
- Wrong AMI IDs
- References to resources that don't exist
- Variables referenced incorrectly
- Any logical errors

> Use `terraform validate` for internal consistency, and `terraform plan` for execution logic.

---

## 2. `terraform init` — Initialize the Project

### What It Does

`terraform init` is the **first command** you must run in any Terraform project. It sets up everything Terraform needs before it can do real work:

| Action | Purpose |
|---|---|
| Creates the `.terraform/` folder | Stores backend and provider info |
| Downloads provider plugins | e.g., the AWS provider (`hashicorp/aws`) |
| Creates `.terraform.lock.hcl` | Locks provider versions for stability across runs |

### When You Must Run It

- Every time you start a **new** Terraform project
- After you **add or change** a provider in your code
- After you **clone** a Terraform project from Git (the `.terraform/` folder is not committed)

### Final Checks Before Running `init`

| Check | Why |
|---|---|
| Inside the correct project folder? | `init` only works in the directory you're in |
| `main.tf` saved? | Unsaved changes won't be read |
| Internet working? | Provider plugins are downloaded from the registry |
| Ran `terraform fmt`? | Optional but recommended |
| Provider block correct? | No red underline / syntax errors |
| AWS credentials configured? | Required for the AWS provider |

### AWS Credentials — Two Options

**Recommended:** Use `aws configure` to set credentials once on your machine:
```bash
aws configure
```
Terraform automatically picks them up.

**Not recommended:** Hardcoding in the `.tf` file:
```hcl
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAxxxxxxx"
  secret_key = "xxxxxxxxxxx"
}
```
This exposes credentials in plain text — never commit this to Git.

### Running It

```bash
terraform init
```

**Expected output:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...

Terraform has been successfully initialized!
```

### Useful Flags

```bash
terraform init -upgrade       # upgrade providers to the latest matching version
terraform init -reconfigure   # ignore existing config, set up backend fresh
```

---

## 3. `terraform validate` — Validate the Code

### What It Does

`terraform validate` checks your code for **syntax and internal consistency** errors. It tells you if your code is structurally and logically valid — without contacting AWS or creating anything.

| Check | Description |
|---|---|
| Syntax check | All `.tf` files are valid HCL |
| Argument validation | Required arguments (like `ami`, `bucket`) are present |
| Error detection | Catches typos, invalid values, misconfigured blocks |
| Offline only | Does NOT verify if AWS resources actually exist |

### When to Run It

| Situation | Run It? |
|---|---|
| After `terraform init` | ✅ Yes |
| After editing `.tf` files | ✅ Yes |
| Before `terraform plan` | ✅ Recommended |
| Before `terraform init` | ❌ Will fail — plugins not downloaded yet |

### Final Checks Before Running `validate`

- `terraform init` has already been run
- `main.tf` and other files are saved
- No syntax errors (no red underlines in your editor)
- All resources have their required arguments

### Running It

```bash
terraform validate
```

**Success output:**
```
Success! The configuration is valid.
```

**Error output:**
```
Error: Missing resource instance name
```

### Key Insight

> `terraform init` prepares your folder.  
> `terraform validate` tells you if your **code** is correct.  
> `terraform plan` tells you if your **logic** is correct against real AWS.

---

## `fmt` vs `validate` — What's the Difference?

| | `terraform fmt` | `terraform validate` |
|---|---|---|
| Purpose | Format/beautify code | Check for syntax & logical errors |
| Modifies files? | Yes (rewrites them) | No |
| Needs `init` first? | No | Yes |
| Catches missing arguments? | No | Yes |
| Catches indentation issues? | Yes (fixes them) | No |
| Catches typos in resource refs? | No | Yes |

**Both are needed.** `fmt` makes code clean. `validate` makes sure it's correct.

---

## Quick Cheat Sheet

```bash
# Standard workflow
terraform fmt          # 1. clean up formatting
terraform init         # 2. initialize provider & backend
terraform validate     # 3. check for errors
terraform plan         # 4. preview changes (next part)
terraform apply        # 5. deploy (next part)
```

---

## Practice Exercises

### Exercise 1 — Run the Full Pre-Deployment Workflow
Create a new folder with a simple `main.tf` containing an AWS provider and an `aws_instance` resource. Run `fmt`, `init`, and `validate` in order. What output do you see from each? What happens if you run `validate` before `init`?

---

### Exercise 2 — Break and Fix
Take a working `main.tf` and intentionally introduce these errors one at a time. Predict which command catches each one:
1. Remove a closing brace `}`
2. Misalign all indentation
3. Reference a variable that doesn't exist: `ami = var.does_not_exist`
4. Use a wrong AMI ID like `"ami-fakefakefake"`

Which errors does `fmt` catch? Which does `validate` catch? Which would only be caught by `plan` or `apply`?

---

### Exercise 3 — `fmt -check` for CI
You're setting up CI to ensure all team members format their code before merging. Which `fmt` flag do you use, and why is `-check` better than running plain `terraform fmt` in CI?

---

### Exercise 4 — Re-initialize
You just added a new provider (e.g., `random`) to your `main.tf`. Do you need to run `terraform init` again? What error would you get if you skipped it and ran `validate` directly?

---

### Exercise 5 — Credentials Setup
A teammate hardcoded AWS credentials inside `provider "aws" { ... }` and committed it to Git. Explain the two problems with this approach, and describe the correct way to handle credentials.

---

*Notes prepared from: Milestone 4 — PDFs 2, 3, 4*  
*`terraform plan`, `apply`, `destroy`, output, and resource referencing follow in later parts.*
