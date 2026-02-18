# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This project implements an Oracle Cloud Infrastructure (OCI) landing zone using Terraform Infrastructure as Code (IaC) managed through HashiCorp Cloud Platform (HCP). The landing zone establishes foundational OCI infrastructure including networking, security, governance, and compute resources with an initial VM instance deployment following OCI Well-Architected Framework principles.

## Project Specifics

| Key              | Value                          |
|------------------|--------------------------------|
| **Name**         | oci-landing-zone-terraform     |
| **Language**     | HCL (HashiCorp Configuration Language) |
| **Framework**    | Terraform                      |
| **Stack**        | OCI, Terraform, HCP Terraform  |
| **Build**        | `terraform plan`               |
| **Test**         | `terraform validate && terraform plan` |
| **Lint**         | `terraform fmt -check`         |
| **Pkg Manager**  | Terraform (modules via registry) |

## Architecture

This project follows a modular Terraform architecture with landing zone design patterns for OCI. Infrastructure is organized into reusable modules for networking, security, compute, and governance. State management is handled through HCP Terraform (formerly Terraform Cloud) for remote state storage, locking, and collaborative workflows.

### Directory Structure

```text
.
├── environments/                    # Environment-specific configurations
│   ├── dev/
│   │   ├── main.tf                 # Dev environment root module
│   │   ├── variables.tf            # Dev-specific variables
│   │   ├── terraform.tfvars        # Dev variable values
│   │   └── backend.tf              # HCP Terraform backend config
│   ├── staging/
│   └── production/
├── modules/                         # Reusable Terraform modules
│   ├── landing-zone/               # Core landing zone module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── networking/                 # VCN, subnets, route tables, NSGs
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── security/                   # IAM policies, compartments, tagging
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/                    # VM instances, instance pools
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── governance/                 # Compartments, tenancy-level controls
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── .terraform/                     # Terraform plugins (gitignored)
├── .terraform.lock.hcl             # Dependency lock file
├── .claude/                        # Orchestration framework
├── terraform.tfvars.example        # Example variable values
├── versions.tf                     # Provider version constraints
└── README.md                       # Project documentation
```

### Key Conventions

**Terraform Module Structure:**
- Each module has `main.tf`, `variables.tf`, `outputs.tf`, and `README.md`
- Use `terraform-docs` to auto-generate module documentation
- Version all module outputs to enable dependent modules

**Naming Conventions:**
- Resources: `<resource_type>-<environment>-<region>-<name>` (e.g., `vcn-dev-us-ashburn-1-hub`)
- Variables: Use snake_case (e.g., `compartment_ocid`)
- Modules: Use kebab-case (e.g., `landing-zone`, `networking`)
- Tags: All resources must include `environment`, `managed_by`, `project` tags

**State Management:**
- Remote state stored in HCP Terraform workspaces
- Each environment has a dedicated workspace
- State locking enabled automatically via HCP
- Sensitive outputs marked with `sensitive = true`

**Variable Management:**
- Define all variables in `variables.tf` with descriptions
- Use `terraform.tfvars` for environment-specific values (gitignored)
- Provide `terraform.tfvars.example` as template
- Store secrets in a secure secrets manager or CI secrets and reference via data sources

**OCI Landing Zone Patterns:**
- Hub-and-spoke VCN topology with DRG connectivity
- IAM policies for governance and access control
- Compartments for organizational hierarchy
- OCI Logging and Monitoring for observability
- Bastion-style access patterns (no public IPs by default)

## Orchestration Protocol

This project uses the **agentic orchestration framework** with specialized agents for workflow management.

### Agent Definitions

#### Builder Agent

- **Purpose**: Implements infrastructure modules, writes Terraform code, creates configuration files
- **Model**: Claude Opus 4.5
- **Permissions**: Read, Write, Edit, Bash
- **When to use**: For all implementation tasks (new modules, resource definitions, variable configurations)

#### Validator Agent

- **Purpose**: Verifies Terraform code correctness, plan output, and compliance with OCI best practices
- **Model**: Claude Opus 4.5
- **Permissions**: Read, Bash (read-only, no Write/Edit)
- **When to use**: After builder completes work, to verify terraform validate passes and plan is safe

#### Test-Runner Agent

- **Purpose**: Runs terraform fmt, validate, plan, and compliance checks (tflint, checkov)
- **Model**: Claude Sonnet 4.5
- **Permissions**: Read, Bash (read-only, no Write/Edit)
- **When to use**: Before committing code, to ensure formatting and validation standards

#### Pre-Flight Agent

- **Purpose**: Validates environment readiness (Terraform installed, OCI CLI authenticated, HCP credentials configured)
- **Model**: Claude Haiku 4.5
- **Permissions**: Read, Bash (read-only, no Write/Edit)
- **When to use**: At session start or before terraform plan/apply operations

### Workflow Pattern

1. **Pre-Flight** → Validate environment (terraform version, oci cli auth, HCP token)
2. **Plan** → Create implementation plan (use `/plan` command)
3. **Build** → Implement via builder agent (use `/build` command)
4. **Validate** → Verify completion via validator agent (terraform validate, plan review)
5. **Test** → Run quality checks via test-runner agent (fmt, validate, tflint)
6. **Review** → Human review terraform plan output before apply
7. **Apply** → Execute terraform apply after approval
8. **Commit** → Git commit when infrastructure changes are applied

## Commands

This project includes custom slash commands:

- `/prime` - Load project context for new sessions
- `/plan` - Create implementation plans for infrastructure changes
- `/build` - Execute builder agent to implement Terraform modules
- `/question` - Answer questions about project structure and OCI architecture

## Hook System

The orchestration framework uses lifecycle hooks for:

- **PreToolUse**: Log tool calls before execution
- **PostToolUse**: Log results, trigger validators on Write/Edit (terraform fmt, terraform validate)
- **PostToolUseFailure**: Log errors for debugging
- **PermissionRequest**: Auto-allow safe operations (terraform fmt, terraform validate, oci commands)
- **SessionStart/End**: Session lifecycle management
- **SubagentStart/Stop**: Agent lifecycle tracking

All hooks are Python scripts executed via `uv run --script` with PEP 723 inline dependencies.

## Guardrails

### Protected Paths

Do NOT modify or delete these paths without explicit permission:

**Terraform State:**
- `*.tfstate` - Local state files (should not exist with HCP backend)
- `*.tfstate.backup` - State backup files
- `.terraform/` - Provider plugins and modules cache
- `.terraform.lock.hcl` - Dependency lock file (commit this)

**Sensitive Files:**
- `terraform.tfvars` - Contains sensitive values (gitignored)
- `.env` - Environment variables with credentials
- `*.pem`, `*.key` - SSH keys and certificates

**OCI Resources:**
- Production environment configurations
- Existing compartments with running workloads
- Network gateways (DRG) and critical VCNs

### Required Permissions

These operations require explicit user permission:

**Destructive Terraform Operations:**
- `terraform destroy` - Destroys all managed infrastructure
- `terraform apply` without reviewing plan first
- `terraform apply -auto-approve` - Bypasses approval
- Modifying production environment configurations
- Changing backend configuration (state migration)

**OCI Resource Changes:**
- Deleting compartments
- Modifying security lists or NSG rules in production
- Changing IAM policies or dynamic groups
- Updating tenancy-wide tag namespaces
- Instance termination or deallocation

**State Management:**
- `terraform state mv` - Moving resources in state
- `terraform state rm` - Removing resources in state
- `terraform import` - Importing existing resources
- Workspace switching in production

### Validation Rules

**Pre-Commit Checks:**
1. Run `terraform fmt -recursive` to format all .tf files
2. Run `terraform validate` to check configuration syntax
3. Run `terraform plan` to preview changes
4. Review plan output for unexpected resource deletions or modifications
5. Verify no sensitive values are hardcoded in .tf files

**Code Quality:**
- All resources must have descriptive names
- All resources must include required tags (environment, managed_by, project)
- Use variables for all configurable values (no hardcoded values)
- Document all modules with README.md and variable descriptions
- Use data sources for existing resources instead of hardcoding IDs

**Security:**
- No public IP addresses without explicit justification
- Network security rules must follow least-privilege principle
- Secrets must not be hardcoded in Terraform files
- Enable OCI Logging and Monitoring for all critical resources
- Use instance principals for automation where possible

## Development Workflow

### Initial Setup

```bash
# Clone repository
git clone <repo-url>
cd oci-landing-zone-terraform

# Install Terraform
# Download from https://www.terraform.io/downloads
# Or use package manager:
# Windows: choco install terraform
# macOS: brew install terraform
# Linux: Follow official instructions

# Install OCI CLI
# Windows: winget install Oracle.OCI-CLI
# macOS: brew install oci-cli
# Linux: bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

# Configure OCI CLI
oci setup config

# Configure HCP Terraform credentials
# Create API token at https://app.terraform.io/app/settings/tokens
# Set environment variable or use terraform login
terraform login

# Initialize Terraform
cd environments/dev
terraform init

# Verify setup
terraform version
oci os ns get
terraform workspace show
```

### Daily Workflow

```bash
# Start session - load context
/prime

# Before making changes - validate environment
terraform version
oci os ns get
terraform workspace show

# Navigate to environment
cd environments/dev

# Pull latest changes
git pull

# Initialize if needed
terraform init -upgrade

# Make infrastructure changes
# Edit .tf files in modules/ or environments/

# Format code
terraform fmt -recursive

# Validate configuration
terraform validate

# Review plan
terraform plan -out=tfplan

# Apply changes (after approval)
terraform apply tfplan

# Commit changes
git add <files>
git commit -m "feat(networking): add subnet for application tier"
git push
```

### Testing

```bash
# Format check (CI/CD)
terraform fmt -check -recursive

# Validate configuration
terraform validate

# Generate and review plan
terraform plan -out=tfplan

# Run tflint (if installed)
tflint --recursive

# Run Checkov security scanning (if installed)
checkov -d . --framework terraform

# Verify specific module
cd modules/networking
terraform init
terraform validate
```

### Linting and Formatting

```bash
# Format all Terraform files
terraform fmt -recursive

# Check formatting without changes
terraform fmt -check -recursive

# Validate configuration syntax
terraform validate

# Generate module documentation (if terraform-docs installed)
terraform-docs markdown table modules/networking > modules/networking/README.md
```

## Stack-Specific Guidelines

### Terraform Best Practices

**Module Design:**
- Keep modules focused and single-purpose
- Use input variables for all configurable aspects
- Output all resource IDs and important attributes
- Include examples/ directory in each module
- Version modules using git tags for production use

**Resource Organization:**
- Group related resources in the same .tf file
- Use descriptive file names (e.g., `network.tf`, `security.tf`, `compute.tf`)
- Separate data sources into `data.tf`
- Keep provider configuration in `providers.tf`
- Define versions in `versions.tf`

**Variable Validation:**
```hcl
variable "environment" {
  description = "Environment name (dev, staging, production)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}
```

**Outputs:**
```hcl
output "vcn_id" {
  description = "The OCID of the VCN"
  value       = oci_core_vcn.hub.id
}

output "admin_password" {
  description = "VM administrator password"
  value       = random_password.admin.result
  sensitive   = true
}
```

### OCI Landing Zone Conventions

**Network Topology:**
- Hub VCN: Central connectivity point (FastConnect/VPN, shared services)
- Spoke VCNs: Workload-specific networks (peered to hub)
- Subnets: Segregate by tier (web, app, data, management)
- NSGs/Security Lists: Apply at subnet and instance level with deny-by-default rules

**Resource Naming:**
```hcl
# Pattern: <resource-type>-<environment>-<region>-<name>
resource "oci_core_vcn" "hub" {
  compartment_id = var.compartment_ocid
  display_name   = "vcn-${var.environment}-${var.region}-hub"
  cidr_blocks    = ["10.0.0.0/16"]
}
```

**Tagging Strategy:**
```hcl
locals {
  common_tags = {
    environment = var.environment
    managed_by  = "terraform"
    project     = "oci-landing-zone"
    owner       = var.owner
    cost_center = var.cost_center
    deployed_at = timestamp()
  }
}

resource "oci_core_vcn" "hub" {
  compartment_id = var.compartment_ocid
  display_name   = "vcn-${var.environment}-${var.region}-hub"
  cidr_blocks    = ["10.0.0.0/16"]
  freeform_tags  = local.common_tags
}
```

### HCP Terraform Integration

**Backend Configuration:**
```hcl
# backend.tf
terraform {
  cloud {
    organization = "your-org-name"

    workspaces {
      name = "oci-landing-zone-dev"
    }
  }
}
```

**Workspace Strategy:**
- One workspace per environment (dev, staging, production)
- Use workspace-specific variable sets in HCP
- Enable remote execution for consistency
- Configure workspace VCS integration for automated runs

**Variable Sets in HCP:**
- Global variables: organization-wide settings
- Workspace variables: environment-specific values
- Sensitive variables: OCI credentials, API keys (marked sensitive)
- Environment variables: OCI_TENANCY_OCID, OCI_USER_OCID, OCI_FINGERPRINT, OCI_PRIVATE_KEY_PATH, OCI_REGION

### VM Instance Configuration

**Compute Module Pattern:**
```hcl
module "vm" {
  source = "../../modules/compute"

  environment     = var.environment
  region          = var.region
  compartment_ocid = var.compartment_ocid
  subnet_id       = module.networking.subnet_ids["app"]

  instance_name   = "vm-${var.environment}-app-001"
  shape           = "VM.Standard.E4.Flex"
  ocpus           = 2
  memory_in_gbs   = 16
  ssh_public_key  = var.ssh_public_key

  source_details = {
    source_type = "image"
    image_id    = var.image_ocid
  }

  enable_monitoring = true
  freeform_tags     = local.common_tags
}
```

**Security Best Practices:**
- Use least-privilege IAM policies for compartments and resources
- Restrict ingress with NSGs and Security Lists
- Prefer private subnets and bastion access patterns
- Enable OCI Logging and Monitoring for instances
- Use instance principals for automation where possible

## Troubleshooting

### Common Issues

**Issue: Terraform initialization fails**
- Symptom: `terraform init` fails with backend configuration errors
- Solution:
  1. Verify HCP Terraform credentials: `terraform login`
  2. Check workspace name in `backend.tf` matches HCP workspace
  3. Ensure organization name is correct
  4. Run `rm -rf .terraform` and `terraform init` again

**Issue: OCI provider authentication fails**
- Symptom: `Error: not authorized` or provider authentication errors
- Solution:
  1. Verify OCI CLI config: `oci setup config` and `oci os ns get`
  2. Confirm required env vars: `OCI_TENANCY_OCID`, `OCI_USER_OCID`, `OCI_FINGERPRINT`, `OCI_PRIVATE_KEY_PATH`, `OCI_REGION`
  3. Check IAM policy permissions for the user or group
  4. Ensure the private key path and fingerprint match the API key

**Issue: Terraform plan shows unexpected resource replacements**
- Symptom: Plan shows resources will be destroyed and recreated
- Solution:
  1. Review what attribute changes triggered replacement
  2. Check if resource supports in-place updates
  3. Use `terraform state` commands to investigate
  4. Consider using `lifecycle { prevent_destroy = true }` for critical resources

**Issue: State lock acquisition failed**
- Symptom: `Error acquiring the state lock`
- Solution:
  1. In HCP Terraform, check if another run is in progress
  2. Cancel stale runs in HCP UI if needed
  3. For local state (not recommended): `terraform force-unlock <lock-id>`
  4. Wait for concurrent operations to complete

**Issue: Module source not found**
- Symptom: `Error: Failed to download module`
- Solution:
  1. Verify module source path is correct (relative paths for local modules)
  2. Run `terraform init -upgrade` to refresh module cache
  3. Check module version constraints in `source` attribute
  4. Ensure git credentials if using git-based modules

**Issue: OCI service limits exceeded**
- Symptom: `LimitExceeded` error during apply
- Solution:
  1. Check limits: `oci limits limit-value list --compartment-id <tenancy_ocid>`
  2. Request limit increase in OCI Console
  3. Use smaller shapes or reduce instance count
  4. Deploy to regions with available capacity

## Environment Requirements

### Required

- **Terraform**: >= 1.9.0 (for HCP Terraform cloud block support)
- **OCI CLI**: >= 3.0.0 (for authentication)
- **Git**: >= 2.30.0 (for version control)
- **HCP Terraform Account**: Free tier or paid (for remote state management)
- **OCI Tenancy**: With appropriate permissions (admins or delegated IAM policies)

### Optional

- **tflint**: >= 0.50.0 (for Terraform linting)
- **Checkov**: >= 3.0.0 (for security scanning)
- **terraform-docs**: >= 0.17.0 (for module documentation)
- **pre-commit**: >= 3.0.0 (for git hooks with automated formatting)

### Environment Variables

**OCI Authentication (API key):**
- `OCI_TENANCY_OCID` - OCI tenancy OCID
- `OCI_USER_OCID` - OCI user OCID
- `OCI_FINGERPRINT` - API key fingerprint
- `OCI_PRIVATE_KEY_PATH` - Path to the API private key file
- `OCI_REGION` - Target region (e.g., `us-ashburn-1`)

**HCP Terraform:**
- `TF_TOKEN_app_terraform_io` - HCP Terraform API token (alternative to terraform login)
- `TF_WORKSPACE` - Workspace name (for CLI-driven workflow)

**Optional:**
- `TF_LOG` - Enable Terraform debug logging (TRACE, DEBUG, INFO, WARN, ERROR)
- `TF_LOG_PATH` - Path to write Terraform logs

## Configuration Files

**Terraform:**
- `versions.tf` - Provider version constraints and required Terraform version
- `providers.tf` - Provider configuration (oci, random, etc.)
- `backend.tf` - HCP Terraform backend configuration
- `main.tf` - Root module resource definitions
- `variables.tf` - Input variable declarations
- `outputs.tf` - Output value definitions
- `terraform.tfvars` - Variable values (gitignored, sensitive)
- `terraform.tfvars.example` - Example variable values template
- `.terraform.lock.hcl` - Dependency lock file (commit to git)

**Orchestration:**
- `.claude/settings.local.json` - Orchestration hooks and permissions
- `.claude/CLAUDE.md` - Minimal project-level instructions
- `CLAUDE.md` - This file (comprehensive project guidance)

**Git:**
- `.gitignore` - Excludes .terraform/, *.tfstate, *.tfvars, .env

## Memory and Context

This project uses three-tier memory:

1. **Auto Memory**: Claude's automatic session memory at `~/.claude/projects/<project>/memory/`
2. **Agent Memory**: Agent-specific persistent memory in `.claude/agent-memory/`
3. **Project Memory**: Shared instructions in `CLAUDE.md` and `.claude/CLAUDE.md`

Use `/prime` at session start to load full project context including:
- OCI landing zone architecture patterns
- Terraform module structure and dependencies
- HCP workspace configuration
- Recent infrastructure changes and plan history

## Special Notes

**State Management Critical:**
- HCP Terraform manages state remotely - NEVER commit *.tfstate files
- State contains sensitive information (passwords, keys, connection strings)
- Use state locking to prevent concurrent modifications
- Back up HCP workspace regularly via API or export

**OCI Cost Management:**
- Use OCI Cost Analysis and Budgets to monitor spending
- Tag all resources for cost allocation and tracking
- Regularly review and deallocate unused resources
- Consider commitment discounts for production workloads

**Security Posture:**
- Follow OCI security best practices
- Enable OCI Logging and Monitoring for all resources
- Use IAM policies and compartments to enforce least privilege
- Regularly review audit logs and security reports

**Compliance and Governance:**
- Use compartments and IAM policies to enforce standards
- Tags enforce cost allocation and ownership
- Audit logs provide traceability
- Resource governance via tenancy-level policies

**Disaster Recovery:**
- Document RTO (Recovery Time Objective) and RPO (Recovery Point Objective)
- Use block volume backups and cross-region replication
- Implement automated backup policies
- Test disaster recovery procedures regularly
- Maintain infrastructure-as-code for rapid rebuilding

---

## Generated by agentic-orchestration framework

This CLAUDE.md was generated using the `/scaffold` command from the agentic-orchestration repository. The orchestration framework provides lifecycle hooks, specialized agents, and workflow automation for Terraform infrastructure development.

**Framework repository**: https://github.com/anthropics/agentic-orchestration
**Documentation**: See `.claude/README.md` for orchestration system details
**Version**: 1.0.0
