---
name: "eks-upgrade-power"
displayName: "EKS Upgrade Readiness"
description: "EKS version upgrade readiness check"
keywords: ["eks", "kubernetes", "upgrade", "k8s upgrade", "eks upgrade", "cluster upgrade", "deprecated api", "karpenter", "addon compatibility", "version skew", "upgrade readiness", "eks version", "node upgrade", "control plane upgrade"]
author: "Thian Kah Haw"
---

# EKS Upgrade Readiness Power

## Overview

This power assesses your live EKS cluster's readiness for a Kubernetes version upgrade. It connects to your cluster via the EKS MCP server, runs automated checks across 8 assessment areas, calculates a readiness score (0-100%), and produces a detailed report with prioritized remediation steps and pre-filled AWS CLI commands.

Unlike the EKS Operation Review Power (which covers broad operational excellence), this power is laser-focused on **upgrade safety** — answering the question: "Is it safe to upgrade this cluster to the next version?"

## What Gets Assessed

| # | Section | Key Checks |
|---|---------|------------|
| 01 | Version Validation | Upgrade path validity, version skew policy, support status |
| 02 | Breaking Changes | Version-specific API removals, behavioral changes, resource impact |
| 03 | Deprecated API Detection | Live scan of cluster resources for deprecated/removed APIs |
| 04 | Add-on Compatibility | Core add-on versions, OSS add-on matrix, Karpenter compatibility |
| 05 | Node Readiness | Node version skew, AL2→AL2023 migration, AMI compatibility |
| 06 | Workload Risks | Single replicas, missing PDBs, health probes, resource requests |
| 07 | AWS Upgrade Insights | Official EKS pre-upgrade checks and recommendations |
| 08 | Upgrade Plan | Pre-filled CLI commands, step-by-step upgrade sequence |

## Readiness Score

The power calculates a weighted readiness score:

| Category | Max Deduction | Rationale |
|----------|--------------|-----------|
| Breaking Changes | 25 pts | Highest risk — can break apps |
| Deprecated APIs | 20 pts | Actionable, fixable pre-upgrade |
| Node Version Skew | 20 pts | Can block upgrade entirely |
| Add-on Compatibility | 15 pts | Critical > optional add-ons |
| Karpenter | 10 pts | Only if installed |
| Workload Risks | 10 pts | Best-practice, not blockers |
| AWS Upgrade Insights | 10 pts | Official AWS checks |
| AL2 Nodes / Behavioral | 10 pts | Informational |

**Score Interpretation:**
- 90-100: **READY** — Safe to proceed
- 80-89: **GOOD** — Minor issues, can proceed with caution
- 70-79: **FAIR** — Several issues need attention first
- 60-69: **RISKY** — Significant issues, not recommended yet
- 0-59: **NOT READY** — Critical blockers, must resolve first

## Onboarding

### Prerequisites

1. **AWS credentials configured** — `aws configure` or `~/.aws/credentials` with EKS access
2. **Python 3.10+** and **uv** installed ([Install uv](https://docs.astral.sh/uv/getting-started/installation/))
3. **kubectl access** to the target cluster (for Kubernetes API queries via MCP)
4. **Required AWS Permissions:**
   - `eks:DescribeCluster`, `eks:ListClusters`, `eks:ListNodegroups`, `eks:DescribeNodegroup`
   - `eks:ListAddons`, `eks:DescribeAddon`, `eks:ListInsights`, `eks:DescribeInsight`
   - `ec2:DescribeSubnets`
   - `iam:GetRole`, `iam:ListAttachedRolePolicies`, `iam:ListRolePolicies`, `iam:GetRolePolicy`

### MCP Server Setup

This power requires two MCP servers configured at the **workspace level** (`.kiro/settings/mcp.json`). See the README for the exact configuration. Do NOT ship a `mcp.json` at the repo root — users configure servers themselves to avoid conflicts with their existing MCP setup.

### Configuration

The MCP servers use your existing AWS credentials. No additional configuration needed if `aws eks list-clusters` works from your terminal.

To use a specific profile or region, users add to the EKS MCP server's `env` in `.kiro/settings/mcp.json`:

```json
"env": {
  "AWS_PROFILE": "your-profile-name",
  "AWS_REGION": "your-region",
  "FASTMCP_LOG_LEVEL": "ERROR"
}
```

### Getting Started

Ask Kiro: *"Run an EKS upgrade readiness assessment"*

The power will discover your clusters, ask which one to assess and what target version, then run the full assessment.

---

## Assessment Workflow

### Step 0: Pre-flight

**Action 1 — List clusters (tests MCP + discovers clusters)**

Call the tool to list EKS clusters. This requires NO cluster name.

- ✅ Success → Show the cluster list. Ask which cluster to assess. If only one cluster, confirm it.
- ❌ Failure → STOP. Do NOT retry more than once. Do NOT read config files. Show:

> **The EKS MCP server isn't responding.** Try these steps:
> 1. Check that MCP servers are configured: `.kiro/settings/mcp.json` should have `awslabs.eks-mcp-server`
> 2. In Kiro: MCP Servers panel → toggle the EKS server off then on
> 3. Check that Python 3.10+ and uv are installed: `uv --version`
> 4. Check that AWS credentials work: `aws sts get-caller-identity`
> 5. Run the permission check: `./tools/check_permissions.sh`

Wait for the user to resolve the issue.

**Action 2 — Describe the selected cluster**

Show: cluster name, Kubernetes version, platform version, region, status, account ID.

**Action 3 — Validate permissions**

After describing the cluster, verify key permissions by attempting:
1. List node groups for the cluster
2. List addons for the cluster
3. Get EKS insights for the cluster

If any fail with AccessDenied, show the user which permission is missing and point them to `./tools/check_permissions.sh <cluster> <region>` for a full check. Do NOT proceed until permissions are confirmed.

**Action 4 — Determine target version**

Ask: *"Your cluster is on v[current]. The next version is v[current+1]. Shall I assess upgrade readiness to v[current+1]?"*

If the user specifies a version more than 1 minor version ahead, explain that EKS requires one-version-at-a-time upgrades and show the required path (e.g., 1.29 → 1.30 → 1.31 → 1.32). Offer to assess the first hop.

**Action 5 — Confirm and proceed**

### Steps 1-8: Run Assessment

Load each steering file in order. For each section:
1. Load the steering file via `readSteering`
2. Execute the checks described in it
3. Collect findings with severity ratings

### Step 9: Calculate Score & Generate Report

Load `report-generation.md` and produce the report.

---

## Tool Usage Rules

1. **Do NOT call any tools when this power is first activated.** Wait for the user to ask.
2. **Do NOT read mcp.json or config files.** Test by calling an actual tool.
3. **Do NOT hardcode or guess cluster names.** Always discover by listing first.
4. **Do NOT retry a failed MCP tool call more than once.**
5. **Always load the relevant steering file before executing checks for that section.**

## Steering File Loading

| User Request | Steering File(s) |
|---|---|
| Full upgrade assessment | ALL files in order |
| Version / upgrade path | `version-validation.md` |
| Breaking changes / API removals | `breaking-changes.md` |
| Deprecated APIs | `deprecated-apis.md` |
| Add-on compatibility / Karpenter | `addon-compatibility.md` |
| Node readiness / AL2 / AMI | `node-readiness.md` |
| Workload risks / PDB / probes | `workload-risks.md` |
| AWS Insights | `upgrade-insights.md` |
| Generate report | `report-generation.md` |

## Report Output

- **Markdown:** `EKS-Upgrade-Assessment-<cluster>-<version>-<YYYY-MM-DD>-<HHMM>.md`
- **HTML:** Run `python3 tools/md_to_html.py <report>.md` to convert

Do NOT generate HTML manually. Always use the conversion script.
