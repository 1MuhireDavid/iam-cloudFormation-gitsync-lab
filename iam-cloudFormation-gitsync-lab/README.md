# IAM Users, Groups & CloudFormation Git Sync Lab

This repo contains a CloudFormation template that stands up the IAM
resources for the lab, deployed via CloudFormation's **Git Sync** feature
(no manual `aws cloudformation deploy` needed once wired up).

## Repo contents


| File | Purpose |
|---|---|
| `template.yaml` | The CloudFormation template — Secrets Manager secret, IAM groups/policies, IAM users, and a custom resource that sets each user's console login profile. |
| `cloudformation.yaml` | The Git Sync **deployment file** CloudFormation reads to know which stack name/region/capabilities/parameters to use when it syncs this repo. |
| `README.md` | This file. |

## What the template creates

1. **Secrets Manager secret** (`OneTimePasswordSecret`) — a 16-character
   auto-generated password shared as the initial console password for all
   three users.
2. **`S3-List-Group`** — allows `s3:ListAllMyBuckets` / `s3:GetBucketLocation` only.
3. **`EC2-Manage-Group`** — allows `ec2:DescribeInstances` (list) and
   `ec2:RunInstances` (create), plus the read calls the EC2 console needs
   to launch an instance (AMIs, key pairs, security groups, subnets, VPCs).
4. **Three IAM users**, each with console access and `PasswordResetRequired: true`:
   - `ec2-user1` → member of `EC2-Manage-Group` (can list + launch EC2 instances)
   - `ec2-user2` → member of `EC2-Manage-Group`, **plus** an inline policy
     with an explicit `Deny` on `ec2:RunInstances`. In IAM, an explicit
     Deny always wins over an Allow, so `ec2-user2` can list EC2 instances
     but cannot launch new ones.
   - `s3-user` → member of `S3-List-Group` (can list S3 buckets only)
5. **A Lambda-backed custom resource** (`Custom::LoginProfile`) that reads
   the secret and calls `iam:CreateLoginProfile` / `iam:UpdateLoginProfile`
   for each user. This step exists because `AWS::IAM::User` has **no**
   native `LoginProfile` property in CloudFormation — setting a console
   password otherwise requires the CLI/console or a custom resource, which
   is what's used here.

## Setting up Git Sync (one-time, console steps)

1. Push this repo to GitHub.
2. In the AWS Console, go to **CloudFormation → Stacks → Create stack →
   "With new resources" → "Sync from Git."**
3. Under **Connect Git repository**, choose **GitHub** and create/select a
   connection (this uses AWS CodeConnections and requires authorizing
   AWS's GitHub app once).
4. Select this repository and the branch to track (e.g. `main`).
5. Set the **deployment file path** to `cloudformation.yaml` (the file
   above) — this is how Git Sync knows the stack name, template path,
   region, capabilities, and parameters.
6. Review the IAM role CloudFormation will create/use to deploy on your
   behalf (least privilege: scope it to the actions this template needs —
   IAM, Secrets Manager, Lambda, CloudFormation, Logs).
7. Create the sync configuration. CloudFormation will immediately deploy
   the current branch HEAD, and will redeploy automatically on every
   future push to that branch (that's the Git Sync security benefit:
   changes are auditable via git history/PR review instead of manual
   console edits, and no long-lived deploy credentials are shared outside
   the CodeConnections integration).

## Retrieving the one-time password

After the stack deploys:

```
aws secretsmanager get-secret-value \
  --secret-id iam-lab/one-time-password \
  --query SecretString --output text
```

Or view it in **Secrets Manager → iam-lab/one-time-password → Retrieve
secret value** in the console.

## Task 2 — testing each user (manual, do this after deploy)

For each of the three users, sign in at the account's IAM sign-in URL
(shown in the stack **Outputs** as `ConsoleSignInUrl`) using the username
and the one-time password above. You'll be forced to set a new password
on first login.

Then, for each user, take two screenshots:

- **`ec2-user1`**
  - EC2 console → Instances list loads (success) → screenshot.
  - EC2 console → Launch instance → completes successfully (success) → screenshot.
- **`ec2-user2`**
  - EC2 console → Instances list loads (success) → screenshot.
  - EC2 console → Launch instance → attempt launch → `UnauthorizedOperation`
    error due to the explicit Deny (failure, as expected) → screenshot.
- **`s3-user`**
  - S3 console → bucket list loads (success) → screenshot.
  - EC2 console → Instances list → `UnauthorizedOperation` / access denied
    (failure, as expected, since `s3-user` isn't in the EC2 group) →
    screenshot.

Save the six screenshots (or however many you choose to include) into a
`screenshots/` folder in this repo and reference them in the PR/lab
write-up, or attach them separately per your lab's submission process.

## Notes / design decisions

- IAM policies are attached inline via `Groups[].Policies` for `S3-List-Group`
  and `EC2-Manage-Group` rather than as separate `AWS::IAM::Policy`
  resources, to keep the group's permission set defined in one place —
  either style is acceptable; separate `AWS::IAM::ManagedPolicy` resources
  would be preferable if the policies needed to be reused or versioned
  independently of the group.
- The explicit Deny on `ec2-user2` is scoped to `ec2:RunInstances` only,
  so all other EC2 group permissions (e.g. `DescribeInstances`) still
  apply — this demonstrates IAM's deny-overrides-allow evaluation logic
  cleanly rather than blocking the whole group's access.
- `CAPABILITY_NAMED_IAM` is required in the Git Sync deployment file
  because the template creates IAM resources with explicit names
  (`ec2-user1`, `ec2-user2`, `s3-user`, the two groups).
