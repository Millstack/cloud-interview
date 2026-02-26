# AWS Networking & VPC

*Focus: Traffic flow and Security boundaries*

## 1. Traffic Routing & Internet Connectivity

### `Qus:` **You have an EC2 instance in a private subnet. How do you ensure it can download security patches from the internet without being exposed to inbound attacks?**

- You must route the instance's outbound traffic (destination `0.0.0.0/0`) to a `NAT Gateway` located in a public subnet, via private `Route Table`
- The NAT Gateway then forwards the request to the `Internet Gateway` (IGW)
- This allows outbound-only communication and the internet cannot initiate a connection back to the private instance

<br>

### `Qus:` **What is the specific difference between a `Public Subnet` and a `Private Subnet` in AWS?**

- Technically, the only difference is the `Route Table`
- A public subnet has a route entry pointing `0.0.0.0/0` (Destination) to an `Internet Gateway` (IGW)
- Whereas a private subnet does not, instead it directs towards NAT gateway

<br>

### `Qus:` **Why would you choose a `NAT Gateway` over a `NAT Instance`?**

- A `NAT Gateway` is a managed, highly available service that scales automatically up to 100 Gbps
- A `NAT Instance` is a single `EC2` instance that you must manage, patch, and manually scale, which creates a single point of failure and a performance bottleneck

<br>

## 2. Advanced Security & Access Control

### `Qus:` **If a Security Group (SG) allows traffic on `Port 80`, but the Network ACL (NACL) denies it, what happens to the request?**

- The request will be `denied`
- Traffic must be allowed by both the SG (`instance level`) and the NACL (`subnet level`) to pass through

<br>

### `Qus:` **Explain the "`Stateful`" nature of Security Groups vs. the "`Stateless`" nature of NACLs**

- Security Groups are `stateful`: if you allow inbound traffic on Port 80, the return outbound traffic is automatically allowed regardless of outbound rules
- NACLs are `stateless`: if you allow inbound traffic, you must also explicitly create an outbound rule to allow the response traffic to leave the subnet

<br>

### `Qus:` **How do you prevent an EC2 instance in a private subnet from using the internet even to reach S3?**

- I would use a `VPC Gateway Endpoint` for S3
- This allows the instance to communicate with S3 using AWS's private internal network instead of traversing the NAT Gateway and the public internet, which also saves on data transfer costs

<br>

### `Qus:` **Can you explain the difference between a `Egress-Only Internet Gateway` and a `NAT Gateway`?**

- An Egress-Only IGW is specifically for `IPv6` traffic, allowing outbound communication while blocking inbound
- A NAT Gateway is typically used for `IPv4` traffic in private subnets to enable outbound-only internet access

<br>

### `Qus:` **How do you resolve a `DNS Resolution` issue for an `EC2` instance trying to reach an internal `RDS` endpoint?**

- I check if `enableDnsSupport` and `enableDnsHostnames` are set to true in the VPC configuration
- If these are disabled, the instance cannot resolve AWS-provided private DNS names

<br>

### `Qus:` **XXXXXXXX**

- XXXXXXXXXXX

<br>

### `Qus:` **XXXXXXXX**

- XXXXXXXXXXX

<br>

## 3. Inter-VPC & Hybrid Connectivity

### `Qus:` **What are the limitations of VPC Peering when connecting multiple VPCs?**

- VPC Peering does not support `transitive routing`
- If VPC A is peered with VPC B, and VPC B is peered with VPC C, VPC A cannot talk to VPC C through VPC B
- You would need a direct peer between A and C, or use a `Transit Gateway`

<br>

### `Qus:` **Can you peer two VPCs that have overlapping CIDR blocks (e.g., both are `10.0.0.0/16`)?**

- No. VPC Peering requires unique, non-overlapping IP address ranges to ensure that traffic is routed to the correct destination

<br>

### `Qus:` **When would you use a `Transit Gateway` instead of `VPC Peering`?**

- I would use a Transit Gateway when managing a large number of VPCs (e.g., 10+) in a `hub-and-spoke model`
- It simplifies management by acting as a central router and supports transitive routing, which VPC Peering does not

<br>

## 4. Troubleshooting & Performance

### `Qus:` **An EC2 instance can't be reached via SSH even though the Security Group allows Port 22. What are the next three things you check?**

- 1. Check the NACL to ensure Port 22 and ephemeral ports are allowed
  2. Verify the instance has a `Public IP` or `Elastic IP` if it's in a public subnet
  3. Check the Route Table to ensure there is a path to the Internet Gateway (`0.0.0.0/0` -> `igw-xxxx`)

<br>

### `Qus:` **How do you monitor the traffic coming in and out of your VPC for security analysis?**

- I would enable VPC Flow Logs
- These logs capture IP traffic information for network interfaces in the VPC and can be published to `CloudWatch` Logs or `S3` for analysis using `Athena`

<br>

### `Qus:` **What is `VPC Flow Logs` and how would you use it in an industrial setting?**

- It captures IP traffic information for network interfaces
- I use it to troubleshoot why traffic is being "REJECTED" — if it's rejected, I know the issue is likely a Security Group or NACL

<br>

### `Qus:` **Your .NET API on EC2 can reach the internet, but it cannot connect to an RDS instance in the same VPC. Both are in private subnets. What is the first thing you check?**

- I check the RDS `Security Group`
- Even if they are in the same VPC, the `RDS Security Group` must explicitly allow `inbound traffic` on the database port from the `Security Group ID` of the EC2 instance
- =Example: `1433 for SQL Server`, `3306 for MySQL`

<br>

### `Qus:` **You’ve confirmed Security Groups are correct, but the connection still times out. What’s next?**

- I verify the `Subnet Routing`
- While traffic within a VPC is local by default, if you are using specific `Custom Route Tables`, ensure there isn't a "Blackhole" route or a conflict preventing the subnets from communicating
- I would also check if the RDS is in a DB Subnet Group that includes the availability zones where my EC2 instances reside

<br>

### `Qus:` **You recently replaced your NAT Gateway with a new one using Terraform or manually. Now, your private instances have lost all internet access. What went wrong?**

- When a NAT Gateway is replaced, its ID changes
- I must verify that the Route Table for the private subnets was updated to point the `0.0.0.0/0` destination to the new `NAT Gateway ID`
- If the Terraform code didn't update the route resource, the traffic is pointing to a non-existent (blackhole) gateway

<br>

### `Qus:` **Your NAT Gateway is working, but your VPC is incurring massive "Data Transfer" costs. How do you investigate?**

- I would enable `VPC Flow Logs` and analyze them using `CloudWatch Insights` or `Athena` to see which private IP is sending the most traffic to the NAT Gateway
- Often, this is caused by a service like S3 or CloudWatch being accessed over the internet
- I would resolve this by implementing a VPC Endpoint to keep that traffic internal and free

<br>

### `Qus:` **An EC2 instance can ping an IP address on the internet, but it cannot resolve `google.com`. What is the issue?**

- This is a DNS resolution failure
- I check if the VPC has `enableDnsSupport` and `enableDnsHostnames` set to `true`

<br>

### `Qus:` **Why do Security Groups not require these "return" rules?**

- Security Groups are `stateful`
- If an inbound request is permitted, the outgoing response is automatically tracked and allowed, regardless of any outbound rules

<br>

### `Qus:` **An EC2 instance in a private subnet can ping another instance in the same VPC but cannot reach the internet. The Route Table shows 0.0.0.0/0 pointing to a NAT Gateway. What do you check next?**

- I check the Public Subnet's Route Table
- The NAT Gateway itself sits in a public subnet, if that public subnet’s route table doesn't have a path to the Internet Gateway (IGW), the NAT Gateway is essentially a "dead end" and cannot forward traffic to the internet

<br>

### `Qus:` **What is a `Blackhole` route in a VPC Route Table, and how does it happen?**

- A blackhole route occurs when a route points to a target that no longer exists
- In an industrial setting using Terraform, this often happens if a resource is deleted manually or via code but the route resource wasn't updated to point to a new target
- Example: A deleted NAT Gateway or VPC Peering connection

<br>

### `Qus:` **Your .NET API is trying to upload a file to S3. Even though you have an S3 Gateway Endpoint configured, the traffic is still going through the NAT Gateway and incurring costs. Why?**

- I check the Route Table associated with the `API's subnet`
- When you create a Gateway Endpoint, you must explicitly select which route tables should be updated
- If the route table doesn't have a prefix list entry for S3 (e.g., `pl-xxxxxx`) pointing to the `vpce-xxxxxx`, the traffic will default to the `0.0.0.0/0` route via the `NAT Gateway`

<br>

### `Qus:` **How do you use VPC Flow Logs to find a "Silent Drop"?**

- I look for records with an `action` of `REJECT`
- If I see a reject, I know the infrastructure (SG or NACL) is dropping it
- If I see ACCEPT but the application still fails, I know the issue is likely within the OS firewall (iptables/firewalld) or the application service itself

