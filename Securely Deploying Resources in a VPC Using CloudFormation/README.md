# Securely Deploying Resources in a VPC Using CloudFormation

A highly available, multi-AZ VPC with a public Apache web tier, a private
application tier, AZ-scoped NAT Gateways, and SSM-only EC2 management (no
SSH, no key pairs).

## Repository contents

| File | Purpose |
|---|---|
| `vpc-ha-secure.yaml` | The complete CloudFormation template (VPC, subnets, routing, NAT, security groups, IAM, EC2, outputs) |
| `README.md` | This file: architecture, deployment steps, validation steps, design notes |

## Architecture at a glance


Key design points:

- **Two AZs, symmetric layout.** Every tier has exactly one resource per AZ, so losing an AZ never takes down the whole tier.
- **One NAT Gateway per AZ, each in its own public subnet, each with its own EIP.** Private route tables are also per-AZ, and each one points only at the NAT Gateway that lives in the *same* AZ. This removes the cross-AZ NAT dependency: if AZ-b's NAT Gateway fails, only AZ-b's private traffic is affected.
- **The public route table is shared** across both public subnets — the route to the Internet Gateway isn't AZ-specific.
- **No SSH anywhere.** Security groups only allow inbound HTTP (public tier) and ICMP (both tiers, for the ping validation step). All access is through SSM Session Manager, using an IAM role with `AmazonSSMManagedInstanceCore` attached via an instance profile. No key pairs are created or referenced anywhere.
- **AMI resolved dynamically** from the public SSM parameter `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64` at deploy time — no hardcoded, region-specific AMI ID.

## Prerequisites

- An AWS account and an IAM principal with permissions to create VPCs, EC2 instances, IAM roles, and CloudFormation stacks.
- AWS CLI v2 installed and configured (`aws configure`), **or** deploy entirely from the console.
- The **Session Manager plugin** for the AWS CLI if starting sessions from your terminal. Not required if using the "Session Manager" button in the EC2 console.

## Step 1 — Deploy the stack

### Option A: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name vpc-ha-secure-lab \
  --template-body file://vpc-ha-secure.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
      ParameterKey=FullName,ParameterValue="Jane Doe" \
      ParameterKey=LabName,ParameterValue="Securely Deploying Resources in a VPC Using CloudFormation"

aws cloudformation wait stack-create-complete --stack-name vpc-ha-secure-lab
aws cloudformation describe-stacks --stack-name vpc-ha-secure-lab --query "Stacks[0].Outputs"
```

`CAPABILITY_NAMED_IAM` is required because the template creates a named IAM role (`SSMInstanceRole`) and instance profile.

### Option B: AWS Console

1. CloudFormation → **Create stack** → With new resources.
2. Upload `vpc-ha-secure.yaml`.
3. Fill in `FullName` and `LabName` parameters (leave networking defaults unless your account already uses `10.0.0.0/16`).
4. Acknowledge that CloudFormation will create IAM resources.
5. Create stack and wait for `CREATE_COMPLETE` (~5–8 minutes; NAT Gateways are the slowest resource to provision).

## Step 2 — Grab the outputs

From the **Outputs** tab (console) or the CLI command above, note:

- `PublicWebServerAUrl`, `PublicWebServerBUrl`
- `PublicWebServerAInstanceId`, `PublicWebServerBInstanceId`
- `PrivateAppServerAInstanceId`, `PrivateAppServerBInstanceId`
- `PrivateAppServerAPrivateIp`, `PrivateAppServerBPrivateIp`

## Step 3 — Validate the web tier

Open both `PublicWebServerAUrl` and `PublicWebServerBUrl` in a browser. Each page shows your name, the lab name, and which AZ served the page.

## Step 4 — Validate connectivity via Session Manager (no SSH)

Start a session on a **public** instance:

```bash
aws ssm start-session --target <PublicWebServerAInstanceId>
```

(Or in the console: EC2 → select the instance → **Connect** → **Session Manager** tab → **Connect**.)

From inside, test public → private reachability:

```bash
ping -c 4 <PrivateAppServerAPrivateIp>
```

Start a second session on a **private** app instance:

```bash
aws ssm start-session --target <PrivateAppServerAInstanceId>
```

Prove outbound-only internet access via NAT (no public IP on this instance):

```bash
ping -c 4 8.8.8.8
traceroute 8.8.8.8      # first public hop should be the NAT Gateway's EIP
sudo dnf install -y tree
```

Repeat both checks against the "B" instances to confirm the second AZ behaves identically and independently.

## Step 5 — Tear down

```bash
aws cloudformation delete-stack --stack-name vpc-ha-secure-lab
```

NAT Gateways and EIPs are billed hourly — delete the stack promptly after your review.

## Design decisions and rubric notes

- **Why no SSH security group rule at all?** Session Manager works entirely over outbound HTTPS from the instance to the SSM service endpoints — it needs no open inbound port, which is why port 22 is omitted entirely rather than just restricted.
- **Why ICMP only from the VPC CIDR / peer security group, not `0.0.0.0/0`?** Ping is included only to prove reachability between tiers during the review; scoping it to internal sources keeps the security groups least-privilege.
- **Why one shared public route table but two private route tables?** A route to an Internet Gateway is identical no matter which AZ a public subnet sits in. A route to a NAT Gateway is not interchangeable across AZs — pointing AZ-b's private subnet at AZ-a's NAT Gateway would create a cross-AZ dependency and defeat the point of having two NAT Gateways.
- **Tagging/naming convention:** every resource is tagged `Name: ${AWS::StackName}-<role>-<az-suffix>`, so the template can be deployed multiple times without name collisions.

## Architectural question: AWS Regional NAT Gateway vs. AZ-scoped NAT Gateway

> Source: AWS "Introducing Amazon VPC Regional NAT Gateway" blog post and the "AWS NAT Gateway now supports regional availability" What's New announcement, both published November 19–20, 2025.

This template uses the traditional, **zonal (AZ-scoped)** NAT Gateway model: one NAT Gateway per AZ, each living in a specific public subnet, each with its own Elastic IP, with per-AZ route tables pointing only at the local NAT Gateway. Adding a third AZ under this model means adding a third public subnet, a third EIP, a third NAT Gateway, and a third private route table — the operational and cost footprint scales linearly with AZ count.

In November 2025, AWS announced a regional availability mode for NAT Gateways, letting a single NAT Gateway automatically expand and contract across the availability zones in a VPC based on where workloads are running, rather than requiring one NAT Gateway per AZ. A Regional NAT Gateway (RNAT) differs from the zonal model in several concrete ways:

- **Scope.** A zonal NAT Gateway is bound to one subnet in one AZ. An RNAT is associated with the entire VPC rather than a specific subnet, so subnet-level configuration doesn't apply to it.
- **Public subnet requirement.** Zonal NAT Gateways must be deployed inside a public subnet. With RNAT, you don't need a public subnet at all to host it — private-subnet-only VPCs become possible, a meaningful security-posture improvement.
- **Scaling behavior.** With the automatic method, the RNAT detects new AZs as workloads appear (via ENI detection) and expands to cover them automatically, reusing the same route table and Gateway ID.
- **Port exhaustion handling.** Each IP address on an RNAT supports up to 55,000 concurrent connections to a unique destination, and RNAT automatically adds additional IP addresses in an AZ (up to 32 per AZ) as that threshold is approached.
- **Route table model.** An RNAT comes with its own AWS-managed route table containing a default route to the VPC's Internet Gateway, which can also be used to insert a Network Firewall or Gateway Load Balancer endpoint into the egress path.

### How RNAT could replace the two zonal NAT Gateways in this lab

1. One `AWS::EC2::NatGateway` resource with its availability mode set to regional (automatic method), associated with the VPC rather than a subnet.
2. **No dedicated public subnets required for NAT hosting** — public subnets would only need to exist for the web tier's own internet-facing needs.
3. **One shared private route table** instead of two, since both private subnets would point at the same Regional NAT Gateway ID.
4. Adding a third AZ later would mean adding a third private subnet and associating it with the *same* existing route table — no new NAT Gateway, EIP, or route table to create.

The trade-off: RNAT does not support the private connectivity type at launch, and the automatic method can take 15–20 minutes (up to 60) to expand into a newly detected AZ, during which traffic is randomly routed to another AZ's RNAT path — a cold-start latency the always-on, per-AZ zonal design here doesn't have. For a small, fixed, two-AZ lab like this one, the two approaches are roughly equivalent in resilience; RNAT's advantage grows with the number of AZs and frequency of AZ expansion.