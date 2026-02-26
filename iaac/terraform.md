# Terraform (Infrastructure as Code)

## 1. Core Workflow & Operations 

### `Qus:` **What is the significance of the terraform.tfstate file, and why is it dangerous to lose it?**

- The state file is the "source of truth" that maps your code to real-world resources
- If lost, Terraform loses track of what it created, and running apply again would attempt to recreate everything, leading to "Resource Already Exists" errors or duplicate infrastructure
- It keeps track of what has been created vs what is updated, in order to no-change, add or delete resources based on changes made

<br>

### `Qus:` **How do you handle a "State Lock" error in a team environment?**

- This usually happens when a previous execution crashed or two people are running apply simultaneously
- After verifying no one else is currently making changes, I would use terraform force-unlock <LOCK_ID> to release the lock in the backend (e.g., DynamoDB)
- to prevent this we deploy state management in S3 bucket with dynamoDB for preventing simultanous appyl commands

<br>

### `Qus:` **What is the difference between `terraform plan -out=tfplan` and a standard `terraform plan`?**

- Using -out saves the execution plan to a file. This is an industrial best practice for CI/CD pipelines because it ensures that the exact plan that was reviewed and approved is the one that gets executed by the apply command later.

<br>

## 2. Modularization & Reusability

### `Qus:` **Why would you choose a "Module" over writing all resources in a single `main.tf`?**

- Modules provide abstraction and reusability
- Industrially, we use them to standardize infrastructure
- It minimize the Blast Radius, for other services mistakely edited
- for example, a "VPC Module" ensures that every environment (Dev, QA, Prod) follows the same subnetting and tagging standards without rewriting code

<br>

### `Qus:` **What is the difference between `count` and `for_each` when creating multiple resources?**

- count is index-based (0, 1, 2)
- If you delete the first item in a list, Terraform may shift the others and force a recreation of resources.
- for_each is map-based and more stable for production because it identifies resources by a unique key rather than an index.

<br>

### `Qus:` **How do you pass information between two different modules?**

- You define an output in the source module and pass it as a variable input to the destination module
- For example, the VPC module outputs the vpc_id, which is then passed into the Security Group module.

<br>

## 3. State Management & Team Logic

### `Qus:` **What is a Remote Backend, and why is it mandatory for industrial projects?**

- A Remote Backend (like S3) stores the state file in a central location rather than on a local machine
- This allows multiple engineers to work on the same infrastructure, provides a version history of the state, and enables state locking

<br>

### `Qus:` **Explain the concept of "`Infrastructure Drift`". How do you fix it?**

- Drift occurs when the real-world infrastructure is changed manually via the AWS Console
- To fix it, I run terraform plan to see the differences
- I can then either run apply to overwrite the manual changes or update my code and use terraform import to bring the manual change into the managed state

<br>

### `Qus:` **How do you manage multiple environments (`Dev`, `Test`, `Staging`, `Prod`) in Terraform?**

- There are two main approaches: Workspace or Directory-based separation
- Workspaces is for managing different states with the same code
- Directory-based separation having a /dev and /prod folder
- Directory-based is often preferred in industry for better isolation and distinct backend configurations

<br>

## 4. Advanced Meta-Arguments

### `Qus:` **What is the `lifecycle` block, and when would you use `prevent_destroy`?**

- The lifecycle block handles resource behavior
- prevent_destroy = true is used for critical production resources like a main Database or a production S3 bucket to prevent accidental deletion during a terraform destroy or apply

<br>

### `Qus:` **When would you use `depends_on`?**

- While Terraform usually handles dependencies automatically, depends_on is used when a resource relies on another resource's behavior that Terraform cannot see
- We use it, when one service is depending upon another service
- Example as an IAM Role being fully propagated before an EC2 instance tries to use it

<br>

## 5. Advanced Scripting & Logic

### `Qus:` **What are "Local Values" (`locals`), and how do they differ from "`Variables`"?**

- Variables are like function arguments—they allow users to pass values into a module
- Locals are like private variables within a module used to store constants or complex expressions (like a combination of tags) so you don't have to repeat the logic multiple times

<br>

### `Qus:` **How do you handle a scenario where you need to create a resource only if a certain condition is met?**

- I would use the Conditional Expression syntax with the count meta-argument
- `count = var.create_resource ? 1 : 0`
- This tells Terraform to create 1 instance if the boolean is true, and 0 (nothing) if it is false

<br>

### `Qus:` **What is a `null_resource`, and why is it used in industry?**

- A `null_resource` is a resource that does nothing in the cloud but allows you to run local scripts or remote commands via provisioners
- It is often used to trigger a configuration script (like a `Bash script`) after a server is provisioned but before it's put into service

<br>

## 6. State Migration & Recovery

### `Qus:` **You manually created an `S3 bucket` in the AWS Console. How do you bring it under Terraform management without deleting it?**

- I use the `terraform import` command. I first write the resource block in my `.tf` file to match the bucket's settings
- Then run terraform import `aws_s3_bucket.my_bucket bucket-name` to link the real resource to my state file

<br>

### `Qus:` **What happens if your `terraform.tfstate` file becomes corrupted?**

- Industrially, we use State Versioning in our S3 bucket backend
- I would simply roll back to the previous version of the state file in S3
- If versioning isn't enabled, I would have to use `terraform import` for every single resource to rebuild the state from scratch

<br>

### `Qus:` **How do you move a resource from one module to another without Terraform destroying and recreating it?**

- I use the `moved` block (introduced in Terraform 1.1)
- This tells Terraform that the resource "Address A" is now "Address B," allowing it to update the state file without touching the actual infrastructure

<br>

## 7. Performance & Optimization

### `Qus:` **If you have 1,000+ resources, `terraform plan` becomes very slow. How do you optimize this?**

- I use the `-refresh=false` flag to skip checking the current state of every resource against the cloud providers
- or I use the `-target` flag to only `plan`/`apply` changes to a specific module or resource

<br>

### `Qus:` **What are "`Sensitive`" variables, and how does Terraform treat them?**

- By marking a variable as `sensitive = true`, Terraform will redact its value in the console output (the logs)
- However, the value is still stored in plain text in the `.tfstate` file, which is why securing the backend (S3/DynamoDB) is critical

<br>

## 8. Provisioners & Lifecycle

### `Qus:` **Why is it generally recommended to avoid `remote-exec` or `local-exec` provisioners?**

- Provisioners do not follow Terraform's "declarative" model
- Terraform can't track what the script actually did
- It is better to use Cloud-Init (`User Data`) or a configuration tool like `Ansible` which are designed for software setup

<br>

### `Qus:` **What is the `create_before_destroy` lifecycle rule?**

- By default, if a resource needs to be replaced, Terraform deletes the old one first
- `create_before_destroy` reverses this, ensuring the new resource is up and running before the old one is killed—crucial for zero-downtime deployments

