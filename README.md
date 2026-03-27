# EKS Upgrade Readiness Power for Kiro

A Kiro Power that assesses your EKS cluster's readiness for a Kubernetes version upgrade. Connects to your live cluster via the AWS-managed EKS MCP server, runs automated checks, calculates a readiness score (0-100%), and generates a detailed report with pre-filled AWS CLI commands.

Upgrade with confidence. Know exactly what will break before you hit the button — deprecated APIs, incompatible add-ons, node version skew, workload risks. No surprises, no rollbacks, no 2 AM pages. Just a clear, prioritized action plan that turns a stressful upgrade into a routine maintenance window.

## Quick Start

### Step 1: Clone this repo

```bash
git clone https://github.com/thiankahaw/eks-upgrade-power.git
```

Open the cloned folder in Kiro as your workspace.

### Step 2: Install MCP Servers

This power requires two MCP servers. You need to configure them at the **workspace level** so they don't interfere with any MCP servers you already have configured globally.

Create the file `.kiro/settings/mcp.json` in this workspace (Kiro may have already created the `.kiro` folder):

```json
{
  "mcpServers": {
    "awslabs.eks-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.eks-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "disabled": false,
      "autoApprove": []
    },
    "awslabs.aws-documentation-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "disabled": false,
      "autoApprove": [
        "search_documentation",
        "read_documentation",
        "read_sections",
        "recommend"
      ]
    }
  }
}
```

> **Why workspace level?** Kiro merges MCP configs with this precedence: user-level (`~/.kiro/settings/mcp.json`) < workspace-level (`.kiro/settings/mcp.json`). If you already have MCP servers configured globally, workspace-level configs are additive — they won't overwrite your existing servers. However, if you already have the EKS MCP server configured globally with custom settings (like a specific `AWS_PROFILE`), the workspace config will take precedence for this workspace only. Your global config remains untouched.

> **Already have these MCP servers?** If you already have `awslabs.eks-mcp-server` and `awslabs.aws-documentation-mcp-server` configured (globally or in another workspace), you can skip this step. Just make sure they're accessible from this workspace.

#### Using a specific AWS profile or region

If your EKS cluster is in a specific account/region, add to the `env` section of the EKS MCP server:

```json
"env": {
  "AWS_PROFILE": "your-profile-name",
  "AWS_REGION": "us-west-2",
  "FASTMCP_LOG_LEVEL": "ERROR"
}
```

### Step 3: Verify Prerequisites

Run the permission check script to validate everything is set up correctly:

```bash
# List available clusters and check basic connectivity
./tools/check_permissions.sh

# Check full permissions against a specific cluster
./tools/check_permissions.sh my-cluster-name us-west-2
```

The script checks:
- AWS CLI installed and credentials valid
- EKS API permissions (list, describe clusters/nodegroups/addons/insights)
- EC2 permissions (describe subnets)
- IAM permissions (list role policies)
- Python 3.10+ and uv installed (required for MCP servers)

### Step 4: Run the Assessment

Open Kiro and ask:

> "Run an EKS upgrade readiness assessment"

The power will:
1. Discover your clusters via the EKS MCP server
2. Ask which cluster and target version
3. Run 8 assessment areas against your live cluster
4. Calculate a readiness score (0-100%)
5. Generate a markdown report with pre-filled CLI commands
6. Optionally convert to HTML via `python3 tools/md_to_html.py <report>.md`

---

## Required AWS Permissions

The IAM principal (user or role) running Kiro needs these permissions. This is a **read-only** policy — the power never modifies your cluster.

### Minimum IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSReadAccess",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:ListNodegroups",
        "eks:DescribeNodegroup",
        "eks:ListAddons",
        "eks:DescribeAddon",
        "eks:DescribeAddonVersions",
        "eks:ListInsights",
        "eks:DescribeInsight",
        "eks:ListAccessEntries",
        "eks:DescribeAccessEntry"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2ReadAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroupRules"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMReadAccess",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:ListAttachedRolePolicies",
        "iam:ListRolePolicies",
        "iam:GetRolePolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

### Kubernetes RBAC

The EKS MCP server also needs Kubernetes API access to list pods, deployments, services, etc. This is handled through your EKS access configuration:

- **EKS API mode (recommended):** Your IAM principal needs an EKS Access Entry with the `AmazonEKSClusterAdminPolicy` or `AmazonEKSAdminViewPolicy` access policy.
- **ConfigMap mode:** Your IAM principal needs to be mapped in the `aws-auth` ConfigMap with a group that has cluster read access.

To verify your Kubernetes access:

```bash
# Update kubeconfig
aws eks update-kubeconfig --name <cluster-name> --region <region>

# Test access
kubectl get nodes
kubectl get pods -A
```

If `kubectl get nodes` works, the EKS MCP server will also work — it uses the same credential chain.

---

## What Gets Assessed

| Section | Checks |
|---------|--------|
| Version Validation | Upgrade path validity, version skew policy, support status & cost |
| Breaking Changes | Per-version API removals, behavioral changes, resource impact |
| Deprecated APIs | Live scan of cluster resources + AWS Upgrade Insights |
| Add-on Compatibility | Core EKS add-ons, OSS add-ons (via matrix), Karpenter |
| Node Readiness | AMI type (AL2→AL2023), container runtime, self-managed nodes |
| Workload Risks | Single replicas, missing PDBs, health probes, resource requests |
| AWS Upgrade Insights | Official EKS pre-upgrade checks and recommendations |
| Upgrade Plan | Pre-filled CLI commands with your cluster name and region |

## Readiness Score

| Score | Level | Meaning |
|-------|-------|---------|
| 90-100 | READY | Safe to proceed |
| 80-89 | GOOD | Minor issues, can proceed with caution |
| 70-79 | FAIR | Several issues need attention first |
| 60-69 | RISKY | Significant issues, not recommended yet |
| 0-59 | NOT READY | Critical blockers, must resolve first |

## Report Output

- **Markdown:** `EKS-Upgrade-Assessment-<cluster>-<current>-to-<target>-<date>.md`
- **HTML:** Run `python3 tools/md_to_html.py <report>.md` (zero external dependencies)

Both files are written to the workspace root.

---

## Project Structure

```
eks-upgrade-power/
├── POWER.md                          # Power definition & agent workflow
├── README.md                         # This file
├── .gitignore
├── steering/                         # Assessment logic (agent instructions)
│   ├── version-validation.md         # Version calendar, upgrade path, skew
│   ├── breaking-changes.md           # Per-version breaking changes (1.25→1.35)
│   ├── deprecated-apis.md            # Deprecated API detection rules
│   ├── addon-compatibility.md        # Core + OSS + Karpenter checks
│   ├── node-readiness.md             # Node group & AMI assessment
│   ├── workload-risks.md             # Workload resilience during upgrade
│   ├── upgrade-insights.md           # AWS official pre-upgrade checks
│   └── report-generation.md          # Score algorithm & report template
├── tools/
│   ├── md_to_html.py                 # Markdown → HTML converter
│   └── check_permissions.sh          # Pre-flight permission validator
└── data/
    └── oss_addon_matrix.json         # OSS add-on compatibility matrix
```

## Troubleshooting

### MCP server not responding

1. In Kiro: open the MCP Servers panel → check if the EKS server shows as "running"
2. Toggle the server off and on
3. Verify prerequisites: `python3 --version` (need 3.10+) and `uv --version`
4. Check credentials: `aws sts get-caller-identity`
5. Test manually: `uvx awslabs.eks-mcp-server@latest`

### Permission denied errors

Run the permission check script:
```bash
./tools/check_permissions.sh <cluster-name> <region>
```

It will tell you exactly which permissions are missing.

### No clusters found

- Check your `AWS_REGION` — clusters are regional
- Check your `AWS_PROFILE` — you may be in the wrong account
- Verify: `aws eks list-clusters --region <region>`

### kubectl works but MCP server can't access Kubernetes API

The MCP server uses the same credential chain as kubectl. If kubectl works but the MCP server doesn't:
- Ensure `AWS_PROFILE` and `AWS_REGION` are set in the MCP server's `env` config
- The MCP server runs in its own process — it doesn't inherit your shell environment

---

## License

Apache 2.0
