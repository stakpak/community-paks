# ConfigHub Usage Guide

## Goals

This rulebook helps you:
- Adopt ConfigHub's Configuration as Data approach for managing Kubernetes deployments
- Manage configuration variants across multiple environments (dev, staging, prod)
- Implement safe change workflows with review, approval, and rollback capabilities
- Eliminate configuration drift between desired and live state
- Automate configuration validation, transformation, and deployment

---

## Key Concepts

### Configuration as Data vs Infrastructure as Code

ConfigHub stores **fully rendered, literal configuration** (WET - Write Every Time) instead of templates with variables. This means:
- No Helm templates, no variable interpolation, no conditionals
- Every environment's config is stored independently in standard YAML
- Changes are made via API/CLI, not git commits and CI/CD pipelines
- Configuration can be queried, validated, and transformed using composable tools

### Core Entities

- **Unit:** A collection of related Kubernetes resources (e.g., Deployment + Service + Ingress)
- **Space:** Logical grouping for units, typically representing an environment or team
- **Revision:** Immutable snapshot of a unit's configuration at a point in time
- **Worker:** Process running in your infrastructure that executes apply/destroy/refresh operations
- **Target:** Represents a deployment destination (Kubernetes cluster)
- **Function:** Code that reads/writes configuration data (readonly, mutating, or validating)
- **Trigger:** Automated function execution on configuration changes (validation, policy enforcement)
- **Link:** Dependency relationship between units for value propagation and apply ordering

---

## Workflow

### Step 1: Install the CLI

Install the `cub` CLI tool:

```bash
curl -fsSL https://hub.confighub.com/cub/install.sh | bash
```

Add to your PATH:

```bash
# Choose one method:
sudo ln -sf ~/.confighub/bin/cub /usr/local/bin/cub
# OR
export PATH=~/.confighub/bin:$PATH
```

**Reasoning:** The CLI is your primary interface for ConfigHub operations. It provides both interactive and scriptable access to all ConfigHub features.

### Step 2: Authenticate and Set Context

Login and configure your default space:

```bash
# Authenticate via browser
cub auth login

# Set default space (replace with your space slug)
cub context set --space <SPACE_SLUG>

# Verify context
cub context get
```

**Reasoning:** Context configuration reduces repetitive flags in subsequent commands.

### Step 3: Create Space Hierarchy for Environments

Organize your environments using spaces with labels:

```bash
# Create a home space for shared entities
cub space create home

# Create environment-specific spaces
cub space create app-dev --label Environment=dev --label Application=myapp
cub space create app-staging --label Environment=staging --label Application=myapp
cub space create app-prod --label Environment=prod --label Application=myapp

# Create platform spaces for shared infrastructure
cub space create platform-dev --label Environment=dev
cub space create platform-prod --label Environment=prod
```

**Reasoning:** Spaces provide isolation boundaries. Labels enable bulk operations across related spaces using where filters.

### Step 4: Set Up Validation Triggers

Install essential validation triggers in your home space:

```bash
# Schema validation (always recommended)
cub trigger create --space home valid-k8s Mutation Kubernetes/YAML vet-schemas

# Placeholder validation (prevents applying incomplete config)
cub trigger create --space home complete-k8s Mutation Kubernetes/YAML vet-placeholders

# Context enforcement (adds metadata to resources)
cub trigger create --space home context-k8s Mutation Kubernetes/YAML ensure-context true
```

Apply triggers to other spaces:

```bash
homeSpaceID="$(cub space get home --jq '.Space.SpaceID')"
cub space create app-dev --label Environment=dev --where-trigger "SpaceID = '$homeSpaceID'"
```

**Reasoning:** Triggers enforce policies automatically. Installing them early prevents invalid configurations from being created or applied.

### Step 5: Deploy and Configure Workers

Create and deploy a worker to enable apply/destroy/refresh operations:

```bash
# Create worker entity
cub worker create --space platform-dev cluster-worker

# Install worker in your cluster
cub worker install cluster-worker \
  --space platform-dev \
  --env IN_CLUSTER_TARGET_NAME=dev-cluster \
  --export --include-secret | kubectl apply -f -

# Verify worker is connected
cub worker list --space "*"
```

**Reasoning:** Workers execute operations in your infrastructure with your credentials. They never send credentials to ConfigHub, maintaining security boundaries.

### Step 6: Import or Create Initial Configuration

**Option A: Import from existing cluster**

```bash
# Import specific resources
cub unit import --space app-dev frontend \
  --namespace myapp \
  --resource-type apps/v1/Deployment \
  --resource-name frontend

# Import entire namespace
cub unit import --space app-dev myapp-all \
  --namespace myapp
```

**Option B: Create from Helm chart**

```bash
# Add Helm repo
helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

# Install chart as ConfigHub unit
cub helm install myapp bitnami/nginx \
  --space app-dev \
  --namespace myapp \
  --values values.yaml
```

**Option C: Create from YAML files**

```bash
# Validate locally first
cub function local vet-schemas deployment.yaml

# Create unit
cub unit create --space app-dev frontend deployment.yaml
```

**Reasoning:** ConfigHub works with existing infrastructure. Import captures current state, while Helm integration leverages the ecosystem. Direct YAML creation works for custom configurations.

### Step 7: Create Configuration Variants

Clone units to create environment variants:

```bash
# Clone single unit to another space
cub unit create --space app-staging \
  --upstream-space app-dev \
  --upstream-unit frontend

# Clone all units from dev to multiple prod spaces
cub unit create --space app-dev \
  --where-space "Labels.Environment = 'prod'"
```

**Reasoning:** Cloning maintains upstream/downstream relationships, enabling controlled propagation of changes while preserving environment-specific overrides.

### Step 8: Manage Dependencies with Links

Create links to propagate values and control apply order:

```bash
# Link backend to namespace (propagates namespace value)
cub link create --space app-dev - backend app-namespace

# Link backend to database (establishes apply order)
cub link create --space app-dev - backend database

# Bulk link all units to namespace
cub link create --space "*" \
  --where-space "Slug = 'app-dev'" \
  --where-from "Slug != 'app-namespace'" \
  --where-to "Slug = 'app-namespace'"
```

**Reasoning:** Links automate value substitution (replacing placeholders) and ensure correct apply/destroy ordering based on dependencies.

### Step 9: Attach Targets to Units

Attach deployment targets to enable apply operations:

```bash
# Attach target to specific unit
cub unit set-target --space app-dev frontend platform-dev/dev-cluster

# Bulk attach to all units in environment
cub unit set-target --space "*" \
  --where "Space.Labels.Environment = 'dev'" \
  platform-dev/dev-cluster
```

**Reasoning:** Targets specify where configuration should be applied. Units without targets cannot be applied.

### Step 10: Make Configuration Changes

Use functions to modify configuration:

```bash
# Update container image
cub function do --space app-dev --unit frontend \
  --change-desc "Update to v1.2.3" \
  set-image-reference main ":v1.2.3"

# Set resource limits
cub function do --space app-dev --unit frontend \
  set-container-resources main all 500m 512Mi 2

# Set environment variable
cub function do --space app-dev --unit frontend \
  set-env-var main DATABASE_URL "postgres://db:5432/myapp"

# Bulk update across environments
cub function do --space "*" \
  --where "Labels.Application = 'myapp' AND Space.Labels.Environment = 'prod'" \
  --change-desc "Production image update" \
  set-image-reference main ":v1.2.3"
```

**Reasoning:** Functions provide type-safe, validated configuration changes. They're more reliable than manual YAML editing and support bulk operations.

### Step 11: Implement Change Management with ChangeSets

Group related changes using ChangeSets:

```bash
# Create ChangeSet
cub changeset create --space home release-v123 \
  --description "Release v1.2.3: Bug fixes and new features"

# Open ChangeSet (add units to it)
cub unit update --patch --space app-prod \
  --filter home/prod-app-filter \
  --changeset home/release-v123 \
  --change-desc "Starting release v1.2.3 rollout"

# Make changes (they'll be part of the ChangeSet)
cub function do --space app-prod \
  --changeset home/release-v123 \
  --filter home/prod-app-filter \
  --change-desc "Update image to v1.2.3" \
  set-image-reference main ":v1.2.3"

# Close ChangeSet
cub unit update --patch --space app-prod \
  --filter home/prod-app-filter \
  --changeset -
```

**Reasoning:** ChangeSets provide atomic, trackable change groups. They enable coordinated rollouts and rollbacks across multiple units.

### Step 12: Review and Approve Changes

If approval triggers are configured:

```bash
# Approve specific revision
cub unit approve --space app-prod frontend --revision 5

# Bulk approve ChangeSet
cub unit approve --space app-prod \
  --filter home/prod-app-filter \
  --revision ChangeSet:home/release-v123
```

**Reasoning:** Approval workflows prevent unauthorized changes to production. They create audit trails for compliance.

### Step 13: Apply Configuration

Apply changes to live resources:

```bash
# Apply single unit
cub unit apply --space app-prod frontend

# Bulk apply with ChangeSet
cub unit apply --space app-prod \
  --filter home/prod-app-filter \
  --revision ChangeSet:home/release-v123

# Dry run to preview changes
cub unit apply --space app-prod frontend --dry-run
```

**Reasoning:** Apply operations are executed by workers in your infrastructure. Links ensure correct ordering. Dry runs enable safe previews.

### Step 14: Manage Configuration Drift

Detect and handle drift between ConfigHub and live state:

```bash
# Refresh to pull live state into ConfigHub
cub unit refresh --space app-prod frontend

# Review differences (compare revisions in UI or CLI)
cub revision list --space app-prod frontend

# Option A: Keep live changes (do nothing)

# Option B: Revert to previous revision
cub unit update --patch --space app-prod frontend \
  --restore "Before:RevisionNum:5"
cub unit apply --space app-prod frontend

# Option C: Merge live changes with pending changes
cub unit update --patch --space app-prod frontend \
  --merge-source Self \
  --merge-base Before:RevisionNum:5 \
  --merge-end RevisionNum:5
```

**Reasoning:** Refresh enables bidirectional sync, unlike GitOps. This accommodates emergency changes and provides transparency.

### Step 15: Propagate Changes Across Variants

Upgrade downstream variants from upstream:

```bash
# Upgrade staging from dev
cub unit update --space app-staging frontend --patch --upgrade

# Bulk upgrade all prod variants
cub unit update --space "*" --patch --upgrade \
  --where "UpstreamUnit.Slug = 'frontend' AND Space.Labels.Environment = 'prod'"

# Preview upgrade changes
cub unit update --space app-staging frontend --patch --upgrade --dry-run
```

**Reasoning:** Upgrade merges upstream changes while preserving downstream overrides. This enables controlled promotion across environments.

### Step 16: Rollback Changes

Rollback using tags or ChangeSets:

```bash
# Tag current state before risky change
cub tag create --space home pre-upgrade-20251030
cub unit tag --space app-prod --revision HeadRevisionNum home/pre-upgrade-20251030

# Rollback to tag
cub unit update --patch --space app-prod \
  --filter home/prod-app-filter \
  --restore "Before:Tag:home/pre-upgrade-20251030"
cub unit apply --space app-prod --filter home/prod-app-filter

# Rollback ChangeSet
cub unit update --patch --space app-prod \
  --filter home/prod-app-filter \
  --restore "Before:ChangeSet:home/release-v123"
cub unit apply --space app-prod --filter home/prod-app-filter
```

**Reasoning:** Tags and ChangeSets provide rollback points. ConfigHub maintains full revision history for point-in-time recovery.

### Step 17: Query and Analyze Configuration

Use functions to extract information:

```bash
# Find all units using specific image
cub unit list --space "*" \
  --resource-type apps/v1/Deployment \
  --where-data "spec.template.spec.containers.*.image#reference = ':v1.2.3'"

# Get container images across environments
cub function do --space "*" \
  --where "Labels.Application = 'myapp'" \
  --output-values-only \
  get-image "*"

# Find placeholders that need replacement
cub function do --space app-dev frontend get-placeholders

# Custom queries with yq
cub function do --space app-dev frontend \
  yq '.[] | select(.kind == "Deployment") | .spec.replicas'
```

**Reasoning:** Configuration as Data enables powerful queries. This supports compliance audits, security reviews, and bulk operations.

### Step 18: Protect Critical Resources

Add gates to prevent accidental deletion:

```bash
# Protect production units
cub unit update --patch --space app-prod \
  --delete-gate critical \
  --destroy-gate critical \
  database

# Protect production space
cub space update --patch --space app-prod \
  --delete-gate production-environment

# Remove gate when needed
cub unit update --patch --space app-prod \
  --delete-gate critical=- \
  database
```

**Reasoning:** Gates prevent accidental destruction of critical infrastructure. They require explicit removal before deletion.

---

## Common Patterns

### Pattern 1: Progressive Rollout

```bash
# Deploy to dev
cub function do --space app-dev set-image-reference main ":v2.0.0"
cub unit apply --space app-dev

# Promote to staging after validation
cub unit update --space app-staging --patch --upgrade
cub unit apply --space app-staging

# Promote to prod regions sequentially
cub unit update --space app-prod-us-east --patch --upgrade
cub unit apply --space app-prod-us-east

# Wait and monitor, then continue
cub unit update --space app-prod-us-west --patch --upgrade
cub unit apply --space app-prod-us-west
```

### Pattern 2: Emergency Hotfix

```bash
# Make direct change in prod
kubectl set image deployment/frontend main=myapp:hotfix -n prod

# Pull change into ConfigHub
cub unit refresh --space app-prod frontend

# Backport to other environments
cub unit update --patch --space app-dev frontend \
  --merge-source app-prod/frontend \
  --merge-base Before:RevisionNum:10 \
  --merge-end RevisionNum:10
```

### Pattern 3: Multi-Environment Configuration

```bash
# Create base unit with placeholders
cub helm install myapp bitnami/nginx --space home

# Clone to environments
cub unit create --space app-dev --upstream-unit home/myapp
cub unit create --space app-prod --upstream-unit home/myapp

# Link to environment-specific dependencies
cub link create --space app-dev - myapp dev-namespace
cub link create --space app-prod - myapp prod-namespace

# Apply environment-specific customizations
cub function do --space app-dev myapp set-replicas 1
cub function do --space app-prod myapp set-replicas 3
```

---

## Troubleshooting

### Apply Gates Blocking Apply

**Symptom:** `cub unit apply` fails with "Apply gate prevents apply"

**Solution:**
```bash
# Check which gates are blocking
cub unit get --space app-prod frontend

# Re-run validation to see details
cub function do --space app-prod frontend vet-placeholders

# Fix issues (e.g., replace placeholders)
cub function do --space app-prod frontend \
  set-namespace production

# Verify gates cleared
cub unit get --space app-prod frontend
```

### Worker Not Connected

**Symptom:** Worker shows `Disconnected` or `Unresponsive`

**Solution:**
```bash
# Check worker status
cub worker list --space "*"

# Check worker pods in cluster
kubectl get pods -n confighub-system

# Check worker logs
kubectl logs -n confighub-system deployment/cluster-worker

# Recreate worker secret if needed
cub worker install cluster-worker --space platform-dev \
  --export-secret-only | kubectl apply -f -
```

### Configuration Validation Failures

**Symptom:** `vet-schemas` trigger fails

**Solution:**
```bash
# Validate locally to see detailed errors
cub function local vet-schemas unit.yaml

# Fix schema issues in configuration
cub unit edit --space app-dev frontend

# Re-validate
cub function do --space app-dev frontend vet-schemas
```

---

## Best Practices

1. **Use Spaces for Isolation:** Create separate spaces for each environment and application combination
2. **Label Everything:** Use consistent labels (Environment, Application, Team) for bulk operations
3. **Install Core Triggers:** Always install `vet-schemas` and `vet-placeholders` triggers
4. **Use ChangeSets for Coordinated Changes:** Group related changes for atomic rollouts and rollbacks
5. **Tag Before Risky Operations:** Create tags before major changes to enable easy rollback
6. **Leverage Links:** Use links for dependency management and value propagation
7. **Protect Production:** Add delete and destroy gates to production resources
8. **Refresh Regularly:** Periodically refresh units to detect drift
9. **Use Filters for Bulk Operations:** Create saved filters for frequently operated-on unit groups
10. **Validate Before Apply:** Use dry-run and local validation before applying changes

---

## References

- [ConfigHub Official Documentation](https://docs.confighub.com/)
- [ConfigHub CLI Reference](https://docs.confighub.com/developer/cli/cub-overview/)
- [Configuration as Data Concept](https://docs.confighub.com/background/config-as-data/)
- [ConfigHub Architecture](https://docs.confighub.com/background/architecture/)
- [Function Authoring Guide](https://github.com/confighub/sdk/blob/main/function/README.md)
- [ConfigHub SDK](https://github.com/confighub/sdk)