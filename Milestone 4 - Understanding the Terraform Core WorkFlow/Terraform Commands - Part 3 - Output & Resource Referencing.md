# Terraform Commands — Part 3: Output & Resource Referencing

> Two related concepts that use the **same syntax**:  
> **Output** → shows resource attribute values *to humans*  
> **Resource Referencing** → passes resource attribute values *between resources*

Both rely on the same pattern: `<resource_type>.<resource_name>.<attribute>`

---

# Part A — Terraform Output

## What Is It?

`output` blocks display useful values after `terraform apply` finishes — like a print statement for your infrastructure.

After Terraform creates resources, AWS assigns dynamic values (instance IDs, public IPs, ARNs, bucket names) that you didn't know in advance. Outputs let you display these values without having to go check the AWS Console.

**Examples of what to output:**
- EC2 → Instance ID, Public IP, Private IP
- VPC → VPC ID, Subnet IDs
- S3 → Bucket name, Bucket ARN

---

## Why Use Output?

- **Quickly see important values** after `apply` — no Console hunting
- **Share values** with other modules or teammates
- **Debug and verify** what Terraform actually created
- **Use values in scripts** (with `terraform output -raw` or `-json`)

---

## Syntax

```hcl
output "<output_name>" {
  value = <resource_type>.<resource_name>.<attribute>
}
```

### What `value` Means

- `value` is the Terraform argument inside the `output` block
- It tells Terraform **what** to display after `apply`
- Whatever you assign to `value =` will be printed in the output

### Common Attributes

| Attribute | What It Returns |
|---|---|
| `.id` | Resource ID — most used (e.g., VPC ID, Subnet ID, Instance ID) |
| `.arn` | Amazon Resource Name |
| `.public_ip` | Public IP of an EC2 instance |
| `.private_ip` | Private IP of an EC2 instance |
| `.bucket` | S3 bucket name |

---

## Basic Example

```hcl
resource "aws_instance" "my_ec2" {
  ami           = "ami-0f918f7e67a3323f0"
  instance_type = "t2.micro"
}

output "ec2_id" {
  value = aws_instance.my_ec2.id
}

output "ec2_public_ip" {
  value = aws_instance.my_ec2.public_ip
}
```

After `terraform apply`:

```
Outputs:

ec2_id        = "i-0abc1234def56789"
ec2_public_ip = "13.235.12.45"
```

---

## Outputting Multiple Values as a Map

You can group related values into one output block by returning a map:

```hcl
output "ec2_details" {
  value = {
    id        = aws_instance.my_ec2.id
    public_ip = aws_instance.my_ec2.public_ip
  }
}
```

- `id` and `public_ip` are **just key names you chose** — not Terraform keywords
- You can name them anything: `instance_id`, `ip_address`, `whatever_you_want`
- The values on the right side are the actual resource attributes

Output:
```
ec2_details = {
  "id"        = "i-0abc1234def56789"
  "public_ip" = "13.235.12.45"
}
```

---

## Optional Fields on `output`

```hcl
output "db_password" {
  value       = aws_db_instance.example.password
  description = "Master password for the RDS instance"
  sensitive   = true   # hides value from CLI output
}
```

| Field | Purpose |
|---|---|
| `description` | Human-readable note about what this output represents |
| `sensitive` | If `true`, value shows as `<sensitive>` in CLI (covered in Milestone 5 Part 7) |

---

## Viewing Outputs Later

After `apply`, you can view outputs anytime:

```bash
terraform output                  # show all outputs
terraform output ec2_public_ip    # show just one
terraform output -raw ec2_id      # raw value (no quotes) — useful in scripts
terraform output -json            # JSON format — useful for automation
```

Example script use:
```bash
EC2_IP=$(terraform output -raw ec2_public_ip)
ssh ec2-user@$EC2_IP
```

---

# Part B — Resource Referencing

When you don't specify everything (VPC, Subnet, SG), how does Terraform handle it? And when you create resources yourself, how do you connect them together? That's resource referencing.

---

## Default Preference — The Mandatory Minimum

For an EC2 instance, only two arguments are truly mandatory:

```hcl
resource "aws_instance" "my_ec2" {
  ami           = "ami-0f918f7e67a3323f0"
  instance_type = "t2.micro"
}
```

So what about VPC, Subnet, Public IP, Security Group? Terraform handles this in three "levels":

---

## Level 1 — Default AWS Setup (No Reference)

When you don't specify VPC or Subnet, AWS uses the **default VPC** for that region.

### What Happens

- AWS automatically places the EC2 in the default VPC
- A subnet is picked from the default VPC
- A public IP is assigned (if auto-assign is enabled)
- The default Security Group is used

### Code

```hcl
resource "aws_instance" "my_ec2" {
  ami           = "ami-0f918f7e67a3323f0"
  instance_type = "t2.micro"
}
```

> Here we're not referencing anything. Terraform just calls AWS, and AWS uses its default resources.

### Catch

If the region has **no default VPC**, EC2 creation fails:

```
Error: No default VPC found for this region
```

Two fixes:
- Create a custom VPC and subnet manually
- Switch to a region that does have a default VPC

---

## Level 2 — Use Existing VPC and Security Group (data sources)

> Note: The PDF mentions Level 2 but doesn't go into detail. This is briefly covered here for completeness.

If a VPC and Security Group already exist (created manually or by another Terraform project), you don't recreate them — you **look them up** using `data` blocks:

```hcl
data "aws_vpc" "existing" {
  id = "vpc-07d5a3b8c1a2f4abc"
}

resource "aws_instance" "my_ec2" {
  ami           = "ami-0f918f7e67a3323f0"
  instance_type = "t2.micro"
  subnet_id     = "subnet-xxx"   # or pulled via data source
}
```

`data` sources are covered separately in later milestones — just know they exist for this case.

---

## Level 3 — Create and Reference (Terraform-to-Terraform)

This is the most common real-world pattern: Terraform creates **everything** — VPC, Subnet, Security Group, EC2 — in one apply.

### Why Referencing Is Needed

When Terraform creates the VPC, AWS generates an ID like `vpc-07d5a3b8c1a2f4abc`. You **don't know this ID in advance** — it's generated at runtime.

But the Subnet needs to know which VPC it belongs to. So how do you pass the VPC ID into the Subnet resource without manually copying it?

That's what **Resource Referencing** does.

### How It Works

Instead of hardcoding the VPC ID, you write:

```hcl
vpc_id = aws_vpc.my_vpc.id
```

This tells Terraform: *"When you create the Subnet, use the ID from the VPC resource I called `my_vpc`."*

### Without Referencing (the painful way)

- Run Terraform to create the VPC
- Manually copy the VPC ID from AWS Console
- Paste it into the Subnet resource as a hardcoded string
- Repeat for SG and EC2 (with Subnet ID and SG ID)
- Slow, error-prone, breaks any kind of automation

### With Referencing (the right way)

| Benefit | What It Means |
|---|---|
| Fully automated | Run `apply` once, everything is built in order |
| No hardcoding | You never type IDs manually |
| Less error risk | No copy-paste mistakes |
| Dynamic updates | If the VPC ID changes, all dependent resources update too |

---

## Full Example — VPC → Subnet → SG → EC2

### Step 1: Create the VPC

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block           = "192.168.0.0/24"
  enable_dns_support   = true
  enable_dns_hostnames = true
}
```

### Step 2: Subnet — Reference the VPC ID

```hcl
resource "aws_subnet" "my_subnet" {
  vpc_id                  = aws_vpc.my_vpc.id   # ← reference
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true
}
```

### Step 3: Security Group — Reference the VPC ID

**Option 1 — Rules inline inside the SG resource:**

```hcl
resource "aws_security_group" "my_sg" {
  name   = "MySecurityGroup"
  vpc_id = aws_vpc.my_vpc.id   # ← reference

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Option 2 — Rules as separate resources:**

```hcl
resource "aws_security_group" "my_sg" {
  name   = "MySecurityGroup"
  vpc_id = aws_vpc.my_vpc.id
}

resource "aws_security_group_rule" "allow_ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.my_sg.id   # ← reference SG ID
}
```

**When to use Option 2:** When you want to manage rules independently — e.g., modify or remove specific rules without touching the SG itself.

### Step 4: EC2 — Reference Subnet ID and SG ID

```hcl
resource "aws_instance" "my_ec2" {
  ami                    = "ami-0f918f7e67a3323f0"
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.my_subnet.id              # ← reference
  vpc_security_group_ids = [aws_security_group.my_sg.id]        # ← reference (in a list)
}
```

> `vpc_security_group_ids` is a **list**, so wrap the reference in `[ ]` — an EC2 can have multiple SGs.

---

## What Can Be Referenced?

Any attribute of a resource, not just `.id`:

| Attribute | Example Use |
|---|---|
| `.id` | Most common — VPC ID, Subnet ID, SG ID |
| `.arn` | Amazon Resource Name |
| `.public_ip` | Pass EC2's public IP to another resource |
| `.private_ip` | Internal IP for routing/DNS |
| `.cidr_block` | Pass a VPC's CIDR to a subnet calculation |

> The same attributes used in `output` blocks are usable in references — that's why these topics belong together.

---

## Implicit Dependency — How Terraform Knows the Order

When you write `vpc_id = aws_vpc.my_vpc.id`, Terraform **automatically** understands:

> "The Subnet depends on the VPC. The VPC must be created first."

This is called **Implicit Dependency** — Terraform builds a dependency graph from references and decides the order on its own.

### Important Behavior

Even if you write the VPC block **after** the Subnet block in your file, Terraform will **still create the VPC first** because of the reference. The order of code in the file doesn't matter — references determine the order of creation.

```hcl
# This works — Terraform still creates the VPC first
resource "aws_subnet" "my_subnet" {
  vpc_id     = aws_vpc.my_vpc.id   # depends on VPC
  cidr_block = "10.0.1.0/24"
}

resource "aws_vpc" "my_vpc" {       # declared after, but built first
  cidr_block = "192.168.0.0/24"
}
```

### Explicit Dependency (`depends_on`)

If a resource has no reference but still depends on another (e.g., due to AWS API behavior), you can force the dependency:

```hcl
resource "aws_instance" "my_ec2" {
  # ... no reference to my_iam_role, but the EC2 needs it to exist first
  depends_on = [aws_iam_role.my_iam_role]
}
```

> Use this sparingly — implicit dependency via references is preferred.

---

## Output and Reference — Side by Side

Both use the same syntax. The difference is **who uses the value**.

| | Output | Reference |
|---|---|---|
| Used in block | `output "name"` | `resource "..." { arg = ... }` |
| Purpose | Show value to the user | Pass value to another resource |
| Syntax | `value = aws_vpc.x.id` | `vpc_id = aws_vpc.x.id` |
| Triggers dependency? | No | Yes (implicit dependency) |
| Visible after apply? | Yes (printed to terminal) | No (used internally) |

---

## Key Points to Remember

- **Output syntax** = **Reference syntax** = `resource_type.resource_name.attribute`
- `.id` and `.arn` are the most common attributes for both
- Outputs are for **humans/scripts**; references are for **other resources**
- References create **implicit dependencies** — Terraform builds the graph automatically
- File order doesn't matter — references control creation order
- Use `depends_on` only when no reference exists to express the dependency

---

## Practice Exercises

### Exercise 1 — Basic Output
You've created an `aws_instance` named `web_server` and an `aws_vpc` named `main_vpc`. Write three `output` blocks to display:
1. The EC2 instance ID
2. The EC2 public IP
3. The VPC ID

What command do you run to view them after `apply`?

---

### Exercise 2 — Map Output
Write a single output block called `server_info` that returns a map with `instance_id`, `public_ip`, and `private_ip` as keys. What's the difference between this and writing three separate output blocks?

---

### Exercise 3 — Implicit Dependency Order
Given this file (written in this exact order):
```hcl
resource "aws_subnet" "subnet1" {
  vpc_id     = aws_vpc.vpc1.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "ec2" {
  subnet_id = aws_subnet.subnet1.id
  ami       = "ami-xxx"
  instance_type = "t2.micro"
}

resource "aws_vpc" "vpc1" {
  cidr_block = "10.0.0.0/16"
}
```
In what order does Terraform actually create these resources? Why?

---

### Exercise 4 — Connecting Resources
Write the Terraform code that:
1. Creates a VPC with CIDR `10.0.0.0/16`
2. Creates a Subnet inside that VPC with CIDR `10.0.1.0/24`
3. Creates a Security Group inside that VPC
4. Launches an EC2 instance using the Subnet ID and Security Group ID

Make sure all the references are correct.

---

### Exercise 5 — Output Public IP for SSH
After applying a Terraform config that creates an EC2, you want to SSH into it. Write the `output` block that gives you the public IP, then write the bash command using `terraform output` to SSH directly.

---

### Exercise 6 — Why a List for SG?
In the EC2 example, why is `vpc_security_group_ids` wrapped in `[ ]` (a list) while `subnet_id` is not? What does this tell you about how many SGs vs subnets an EC2 can have?

---

*Notes prepared from: Milestone 4 — PDFs 8 & 9*  
*This completes the Milestone 4 commands series (Parts 1–3).*
