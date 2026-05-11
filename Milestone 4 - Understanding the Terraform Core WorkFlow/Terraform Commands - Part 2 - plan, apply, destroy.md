# Terraform Commands — Part 2: `plan`, `apply`, `destroy`

> The three commands that **do** the work. Preview changes, apply them, and tear it all down when you're finished.

---

## The Execution Workflow

```
1. terraform plan       →  preview what will happen (safe, no changes)
2. terraform apply      →  actually create/update/delete resources
3. terraform destroy    →  remove everything when done
```

> Part 1 covered the "prepare and verify" commands (`fmt`, `init`, `validate`). This part covers the commands that interact with your real cloud account.

---

## 1. `terraform plan` — Preview Changes (Dry Run)

### What It Does

`terraform plan` is a **preview command**. It compares your `.tf` code with the existing infrastructure (and state file) and tells you exactly what will be **added, changed, or destroyed** — without actually doing it.

Think of it as a dry run: Terraform shows the "diff" between what you want and what currently exists.

### Why Use It

| Benefit | What It Means |
|---|---|
| **Safe testing** | See what Terraform is about to do before any real change happens |
| **Avoid mistakes** | Catch problems before they hit production |
| **Detect drift** | Shows the difference between your code and the actual cloud state |

### When to Run It

- After you've written or edited `.tf` files
- After `terraform init` has run successfully
- Before every `terraform apply` — always

### Example

`main.tf`:
```hcl
resource "aws_instance" "myec2" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

Run:
```bash
terraform plan
```

Output:
```
Terraform will perform the following actions:

  # aws_instance.myec2 will be created
  + resource "aws_instance" "myec2" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t2.micro"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### How to Read the Output

| Symbol | Meaning |
|---|---|
| `+` | Resource will be **created** |
| `~` | Resource will be **modified** |
| `-` | Resource will be **destroyed** |
| `-/+` | Resource will be **replaced** (destroyed and recreated) |

The last line — `Plan: X to add, Y to change, Z to destroy` — is the summary you should always read carefully.

### Important

> `terraform plan` **does not make any changes**. It only shows what *would* happen.  
> You must run `terraform apply` to actually create the resources.

### Useful Flags

```bash
terraform plan -out=tfplan       # save the plan to a file (for use with apply later)
terraform plan -destroy          # preview what a destroy would do
terraform plan -target=aws_instance.myec2   # plan changes for a specific resource only
```

---

## 2. `terraform apply` — Execute the Changes

### What It Does

`terraform apply` takes the plan and actually applies it — it creates, updates, or deletes resources in your cloud account based on your `.tf` files.

### Why Use It

| Action | What It Does |
|---|---|
| **Create** | Deploys new resources (EC2, VPC, S3, etc.) |
| **Update** | Applies config changes you made to existing resources |
| **Remove** | Deletes resources that you removed from `.tf` files |

### When to Run It

- After `terraform plan` ran successfully
- After you've **reviewed** the planned changes
- When you're ready to make real changes to your AWS account

### What Happens When You Run It

```bash
terraform apply
```

Terraform will:

**1. Show the plan output again** — for your final review.

**2. Ask for confirmation:**
```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

**3. You type one of:**
- `yes` → Apply proceeds, resources are created/updated/destroyed
- Anything else (including `no`) → Apply is cancelled, nothing changes

**4. On success:**
```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### What Happens Internally

- Resources are created, modified, or destroyed in your cloud account
- The `terraform.tfstate` file is updated to reflect the new "current state"
- Terraform now knows what exists so it can compute the next diff

### If You Rerun `apply` Without Changing Code

Nothing happens — Terraform sees no diff between your code and the current state.

```
No changes. Your infrastructure matches the configuration.
```

### Useful Flags

```bash
terraform apply -auto-approve       # skip the yes/no prompt — use in CI/CD only
terraform apply tfplan              # apply a previously saved plan (from -out=tfplan)
terraform apply -target=aws_instance.myec2   # apply only to a specific resource
```

> `-auto-approve` is dangerous in interactive use. It's there for automation pipelines where a human already reviewed the plan.

---

## 3. `terraform destroy` — Tear It All Down

### What It Does

`terraform destroy` removes **all resources** managed by your Terraform project. It tears down the infrastructure defined in your `.tf` files and updates the state file to reflect that nothing exists.

### Why Use It

| Reason | Use Case |
|---|---|
| **Clean up** | After testing or experimenting |
| **Avoid cost** | AWS charges for running resources — destroy when done |
| **Reset** | Wipe everything and start fresh |

### When to Run It

- You've finished testing/development
- You no longer need the infrastructure
- You want to make sure no leftover resources are running and charging

### What Happens When You Run It

```bash
terraform destroy
```

Terraform will:

**1. Show all resources it plans to destroy** (everything marked with `-`).

**2. Ask for confirmation:**
```
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

**3. You type:**
- `yes` → All resources are deleted
- Anything else → Destroy is cancelled

**4. On success:**
```
Destroy complete! Resources: 1 destroyed.
```

### What Happens Internally

- Resources are **deleted** from your cloud provider
- The `terraform.tfstate` file is updated — it now reflects an empty environment
- Terraform knows the environment is clean

### Useful Flags

```bash
terraform destroy -auto-approve              # skip the yes/no prompt
terraform destroy -target=aws_instance.myec2 # destroy only a specific resource
```

> Be very careful with `destroy` — there is **no undo**. Once a resource is deleted, you can't recover it (data on EBS volumes, S3 objects, RDS databases, etc. will be lost unless you have backups).

---

## The Full Workflow — End to End

```bash
# Setup phase (Part 1)
terraform fmt
terraform init
terraform validate

# Execution phase (Part 2)
terraform plan          # 1. preview
terraform apply         # 2. deploy → type yes
# ... use your infrastructure ...
terraform destroy       # 3. tear down → type yes
```

---

## `plan` vs `apply` vs `destroy` — Side by Side

| | `plan` | `apply` | `destroy` |
|---|---|---|---|
| Makes real changes? | ❌ No | ✅ Yes | ✅ Yes |
| Needs confirmation? | ❌ No prompt | ✅ Yes (type `yes`) | ✅ Yes (type `yes`) |
| Updates state file? | ❌ No | ✅ Yes | ✅ Yes |
| Reversible? | N/A | Can be re-applied to fix | ❌ No undo |
| Use case | Preview before doing anything | Create/update infra | Tear down infra |

---

## Key Rules to Remember

- Always run `plan` before `apply` — read the output, don't skip it
- Only `yes` confirms `apply` and `destroy` — anything else cancels
- Re-running `apply` without code changes does nothing (no diff)
- `destroy` is final — there's no undo button
- `-auto-approve` is for automation only — never in interactive use
- The state file is updated after every `apply` and `destroy` — don't lose it

---

## Practice Exercises

### Exercise 1 — Read the Plan Output
You run `terraform plan` and see this:
```
Plan: 2 to add, 1 to change, 3 to destroy.
```
What does this tell you? Should you proceed with `apply` without looking further? Why or why not?

---

### Exercise 2 — Cancel an Apply
You run `terraform apply`, review the plan, and realize one of the changes is wrong. At the `Enter a value:` prompt, what should you type to safely cancel? What happens if you accidentally hit Enter?

---

### Exercise 3 — Re-Apply Behavior
You ran `terraform apply` successfully and created an EC2 instance. Without changing any code, you run `terraform apply` again. What happens, and why?

---

### Exercise 4 — Saved Plan
You want to ensure that the plan reviewed by your team lead is exactly what gets applied. Use the `-out` flag to save the plan, then apply that exact plan. Write the two commands.

---

### Exercise 5 — Targeted Destroy
You have a project with three resources: an EC2 instance, a VPC, and an S3 bucket. You only want to destroy the S3 bucket, not the others. Write the correct `destroy` command. Why might using `-target` be considered a "last resort" in Terraform?

---

### Exercise 6 — Cost Cleanup
You ran a Terraform project last week with some test resources but forgot to destroy them. A week later you realize AWS has been charging you. You go back to the same project folder and run `terraform destroy`. Will it know what to delete? What file makes this possible?

---

*Notes prepared from: Milestone 4 — PDFs 5, 6, 7*  
*Terraform Output and Resource Referencing follow in Part 3.*
