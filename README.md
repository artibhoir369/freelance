# DevOps Engineer — Interview Guide & Skills Documentation

## Table of Contents

1. [Role Summary](#1-role-summary)
2. [Infrastructure-as-Code (CloudFormation)](#2-infrastructure-as-code-cloudformation)
3. [Deployment Automation & Scripting](#3-deployment-automation--scripting)
4. [Containerization & CI/CD](#4-containerization--cicd)
5. [AWS Services Expertise](#5-aws-services-expertise)
6. [Database Management](#6-database-management)
7. [Configuration Management](#7-configuration-management)
8. [Security & IAM](#8-security--iam)
9. [Monitoring, Logging & Troubleshooting](#9-monitoring-logging--troubleshooting)
10. [Shell Scripting Patterns & Techniques](#10-shell-scripting-patterns--techniques)
11. [Key Commands Cheat Sheet](#11-key-commands-cheat-sheet)
12. [Interview Q&A](#12-interview-qa)

---

## 1. Role Summary

**What I do:** I design, build, and maintain the end-to-end deployment infrastructure for a multi-tier AWS application stack. I own the full lifecycle — from provisioning 40+ CloudFormation stacks to deploying containerized AI agents, managing databases, and automating frontend builds.

**Scale:**
- 40+ CloudFormation stacks deployed in strict dependency order
- 22 Lambda functions
- 2 Bedrock AgentCore AI agent containers
- 10+ DynamoDB tables
- Redshift Serverless warehouse (10 tables + 9 materialized views)
- 3 API Gateway types (REST, HTTP, WebSocket)
- Multi-region deployment (us-east-1, us-west-2)
- One-click deployment script (3200+ lines) that provisions everything in 18-25 minutes

---

## 2. Infrastructure-as-Code (CloudFormation)

### Template Organization

I organize 40+ CFN templates into 8 logical domains by lifecycle and dependency:

```
infra/
├── 01-networking/       # VPC, Subnets, Cognito (auth layer)
├── 02-storage/          # S3 buckets, EventBridge→Firehose pipeline
├── 03-database/         # DynamoDB, Redshift Serverless, Timestream
├── 04-messaging/        # SNS topics
├── 05-compute-lambdas/  # 22 Lambda function stacks
├── 06-api-gateway/      # REST API Gateway (50+ routes)
├── 07-ml-sagemaker/     # Glue, Feature Store, SageMaker Endpoint, Studio
└── 08-ui-frontend/      # S3 + CloudFront hosting
```

### Key Patterns I Use

**1. Parameterized templates for reusability:**
- Use-case prefix, region, bucket names all parameterized
- Same templates deploy to multiple environments/regions

**2. Conditional resources:**
```yaml
Conditions:
  HasCustomDomain: !Not [!Equals [!Ref DomainName, '']]

ViewerCertificate: !If
  - HasCustomDomain
  - AcmCertificateArn: !Ref CertificateARN
    SslSupportMethod: sni-only
  - CloudFrontDefaultCertificate: true
```

**3. Cross-stack output passing:**
- Read CFN stack outputs at deploy time
- Inject as parameters to downstream stacks
- Eliminates hardcoded references between stacks

**4. DeletionPolicy: Retain for stateful resources:**
```yaml
WebsiteBucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain
  UpdateReplacePolicy: Retain
```

**5. Large template handling:**
- Templates >51200 bytes → upload to S3 → deploy via `--template-url`
- Smaller templates → direct `--template-body file://`

**6. Parameter filtering:**
- Inline Python parses template YAML to find declared parameters
- Only passes parameters that exist in the template (avoids CFN validation errors)
- Handles both JSON and YAML template formats

### Deployment Dependency Management

I manage deployment order across 39 phases with explicit dependency chains:

```
Phase 1:  S3 bucket (no deps)
Phase 2:  VPC (no deps)
Phase 3:  DynamoDB [parallel: 4 stacks] (no deps)
Phase 4:  Redshift Serverless (depends on VPC)
Phase 5:  Cognito (no deps)
Phase 7:  HTTP API Lambda (depends on S3)
Phase 8:  4 Lambdas [parallel] (depends on Phase 7 — HTTP API ID)
...
Phase 35.5: REST API Gateway (depends on ALL Lambda phases — needs all ARNs)
Phase SSM: Parameter Store seeding (depends on all stacks)
Phase UI:  Frontend build + deploy (depends on SSM)
```

### Template Validation

I validate all templates before deployment using two methods:

1. **API validation:** `aws cloudformation validate-template`
2. **Static analysis:** Flag hardcoded account IDs and region strings outside of `!Sub` and `Mappings` contexts

```bash
# Validation script checks:
# - CFN API syntax validation (all templates)
# - Large templates validated via S3 upload
# - Hardcoded account IDs flagged
# - Hardcoded region strings flagged
# - Throttled responses handled gracefully (non-fatal)
./scripts/validate_cfn.sh --region us-west-2
```

---

## 3. Deployment Automation & Scripting

### One-Click Orchestrator — How I Built It (3200+ lines Bash)

#### Why I Built It

The platform has 40+ interdependent CloudFormation stacks. Deploying manually via AWS Console or individual CLI commands was:
- Error-prone (wrong order breaks cross-stack references)
- Time-consuming (45+ min manually vs 18 min automated)
- Non-repeatable (different engineers deploy differently)
- Risky (no automatic rollback on failure)

I designed and built a single Bash orchestrator that anyone on the team can run with one command to provision or update the entire environment.

#### Skills Used to Build It

| Skill | How It's Applied |
|-------|-----------------|
| **Advanced Bash scripting** | 3200+ lines — functions, traps, arrays, subshells, process management, signal handling |
| **AWS CLI mastery** | 20+ AWS services called programmatically (CFN, S3, ECR, CodeBuild, SSM, IAM, Cognito, Redshift, Lambda, CloudFront, AgentCore) |
| **Python (inline)** | Embedded Python for JSON/YAML parsing, API response processing, regex extraction — Bash alone can't handle complex data structures |
| **CloudFormation deep knowledge** | Stack lifecycle states, changeset behavior, EarlyValidation hooks, large-template handling, capability flags |
| **Distributed systems thinking** | Dependency ordering, parallel execution, eventual consistency handling (IAM propagation delays, DNS propagation) |
| **Error engineering** | Retry patterns, exponential backoff, dead-letter detection, tombstone cleanup, transient vs terminal failure classification |
| **Docker** | Multi-platform builds (ARM64), buildspec generation, ECR authentication, container lifecycle |
| **Security awareness** | Temp file permissions (chmod 600), deny-group handling, credential passthrough, secret masking |
| **Idempotency design** | Every operation is safe to re-run — checks state before acting, skips already-done work |
| **Observability** | Progress logging, elapsed timers, audit trail files, tee output capture |

#### Architecture of the Script

```
┌─────────────────────────────────────────────────────────┐
│  ARGUMENT PARSING & CONFIG                               │
│  --region, --stack, --dry-run, --numpy-layer-arn         │
│  Derive all resource names from USE_CASE prefix          │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  PRE-FLIGHT CHECKS (Phase 0)                             │
│  Tools, credentials, artifacts, services reachable       │
│  Fail fast with clear error messages                     │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  HELPER FUNCTIONS                                        │
│  validate_template()    — CFN API syntax check           │
│  stack_needs_deploy()   — Status-aware skip/proceed      │
│  deploy_stack()         — Retry wrapper (3 attempts)     │
│  deploy_stack_parallel()— Background subshell + log      │
│  wait_parallel()        — Collect parallel results       │
│  get_output()           — Read CFN stack outputs         │
│  rollback()             — ERR trap handler               │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  PARAMETER FILE GENERATION (parms.json)                  │
│  Dynamic JSON with all resource names derived from       │
│  USE_CASE, REGION, ACCOUNT_ID                            │
│  Region-specific values (SageMaker container URIs)       │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  PHASED DEPLOYMENT (39 phases)                           │
│  Sequential where dependencies exist                     │
│  Parallel where stacks are independent                   │
│  Each phase: validate → check state → deploy/skip        │
│  Cross-stack outputs read and injected downstream        │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  POST-DEPLOY AUTOMATION                                  │
│  SSM seeding, VPC DNS enable, Cognito URL update,        │
│  UI build + deploy, CloudWatch log retention             │
└─────────────────────────────────────────────────────────┘
```

#### How Each Core Feature Works

---

##### 1. Pre-Flight Validation (Fail Fast)

**Principle:** Never start deploying if prerequisites are missing. Fail immediately with a clear error.

```bash
# Version checking with semver comparison:
CURRENT_AWS_VERSION=$(aws --version 2>&1 | awk '{print $1}' | cut -d'/' -f2)
if [[ "$(printf '%s\n' "$MIN_VERSION" "$CURRENT" | sort -V | head -n1)" != "$MIN_VERSION" ]]; then
  echo "❌ AWS CLI $CURRENT is below required $MIN_VERSION"
  exit 1
fi

# Service reachability (not just credentials):
aws cloudformation list-stacks --region "$REGION" --max-items 1 &>/dev/null || exit 1

# Artifact completeness (check ALL required files up front):
_MISSING=()
for _req_zip in "lambda-a.zip" "lambda-b.zip" "lambda-c.zip"; do
  [[ ! -f "${CODE_DIR}/${_req_zip}" ]] && _MISSING+=("$_req_zip")
done
[[ ${#_MISSING[@]} -gt 0 ]] && { echo "❌ Missing: ${_MISSING[*]}"; exit 1; }

# Parameter format validation (regex):
if [[ ! "$ARN" =~ ^arn:aws:lambda:[a-z0-9-]+:[0-9]{12}:layer:[A-Za-z0-9_-]+:[0-9]+$ ]]; then
  echo "❌ Invalid ARN format"; exit 1
fi
```

**Skills demonstrated:** Input validation, regex, version comparison (`sort -V`), array accumulation for batch error reporting.

---

##### 2. Stack State Machine (Idempotent Deploys)

**Principle:** The script must be safe to re-run at any point. It observes current state and decides the action.

```bash
stack_needs_deploy() {
  local STACK_NAME="$1"
  local STATUS
  STATUS=$(aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" --region "$REGION" \
    --query "Stacks[0].StackStatus" --output text 2>/dev/null || echo "DOES_NOT_EXIST")

  case "$STATUS" in
    CREATE_COMPLETE|UPDATE_COMPLETE|UPDATE_ROLLBACK_COMPLETE)
      echo "  ⏭️  Already $STATUS — skipping"
      return 1  # Signal: skip
      ;;
    DOES_NOT_EXIST|"")
      return 0  # Signal: proceed with create
      ;;
    *ROLLBACK*|*FAILED)
      # Tombstone: delete first, then proceed with fresh create
      aws cloudformation delete-stack --stack-name "$STACK_NAME" --region "$REGION"
      # Poll until actually gone (can take 10+ min for Redshift):
      local _poll=0
      while [[ $_poll -lt 900 ]]; do
        local _s=$(aws cloudformation describe-stacks ... || echo "DOES_NOT_EXIST")
        [[ "$_s" == "DOES_NOT_EXIST" ]] && break
        [[ "$_s" == "DELETE_FAILED" ]] && {
          # Retry with --retain-resources for stuck resources
          aws cloudformation delete-stack --retain-resources Resource1 Resource2 ...
        }
        sleep 30; _poll=$((_poll + 30))
      done
      return 0
      ;;
    REVIEW_IN_PROGRESS)
      # Stale changeset from aborted deploy — clear it
      aws cloudformation delete-stack --stack-name "$STACK_NAME" --region "$REGION"
      aws cloudformation wait stack-delete-complete ...
      return 0
      ;;
    *IN_PROGRESS)
      # Another process is deploying — wait for it to settle
      aws cloudformation wait stack-create-complete ... || \
      aws cloudformation wait stack-update-complete ... || true
      return 0
      ;;
  esac
}
```

**Skills demonstrated:** State machine design, CFN lifecycle understanding, polling with timeout, graceful handling of all edge cases.

---

##### 3. Deploy with Retry (Resilient Execution)

**Principle:** Transient failures (throttling, IAM propagation) should be retried. Terminal failures should stop immediately.

```bash
deploy_stack() {
  local STACK_NAME="$1" TEMPLATE_FILE="$2" PARAMS_FILE="${3:-}"
  
  # Skip if doesn't match --stack filter
  [[ -n "$STACK_FILTER" && "$STACK_NAME" != *"$STACK_FILTER"* ]] && return 0
  
  # Validate template syntax first
  validate_template "$TEMPLATE_FILE"
  
  # Check if already deployed (idempotent)
  stack_needs_deploy "$STACK_NAME" || return 0
  
  local ATTEMPT=0 SUCCESS=false
  while [[ $ATTEMPT -le $MAX_RETRIES ]]; do
    [[ $ATTEMPT -gt 0 ]] && { sleep $(( ATTEMPT * 15 )); }  # Exponential backoff
    ATTEMPT=$(( ATTEMPT + 1 ))
    
    # Handle large templates (>51KB) via S3
    local FILE_SIZE=$(wc -c < "$TEMPLATE_FILE")
    if [ "$FILE_SIZE" -gt 51200 ]; then
      aws s3 cp "$TEMPLATE_FILE" "s3://${BUCKET}/cfn-templates/$(basename $TEMPLATE_FILE)"
      TEMPLATE_ARG="--template-url https://s3.amazonaws.com/${BUCKET}/cfn-templates/..."
    else
      TEMPLATE_ARG="--template-body file://${TEMPLATE_FILE}"
    fi
    
    # Filter parameters to only those declared in this template (Python inline):
    PARAM_ARGS=$(python3 -c "
import json, re, sys
params = json.load(open('$PARAMS_FILE'))
# Parse template to find declared parameter keys (handles YAML + JSON)
# Only emit parameters that exist in the template
..." 2>/dev/null)
    
    # New stack: create-stack (avoids EarlyValidation changeset hook)
    # Existing stack: cloudformation deploy (changeset-based update)
    if [[ "$EXISTING_STATUS" == "DOES_NOT_EXIST" ]]; then
      aws cloudformation create-stack --capabilities CAPABILITY_NAMED_IAM ...
      aws cloudformation wait stack-create-complete ...
    else
      aws cloudformation deploy --no-fail-on-empty-changeset ...
    fi
    
    # On failure — classify error type:
    case "$STACK_STATUS" in
      *ROLLBACK_COMPLETE|*FAILED)
        # Check if EarlyValidation (resource exists) — recoverable
        if grep -q "already exists" <<< "$EVENTS"; then
          # Delete tombstone + retry (CFN will adopt existing resources)
          aws cloudformation delete-stack ... && ATTEMPT=0 && continue
        else
          break  # Terminal — don't retry
        fi
        ;;
      *IN_PROGRESS)
        # Wait for concurrent operation to finish, then retry
        ;;
    esac
  done
  
  # Track successful deployment for rollback
  echo "$STACK_NAME" >> "$OUT_STACKS_FILE"
}
```

**Skills demonstrated:** Retry patterns, failure classification, exponential backoff, large-file handling, dynamic parameter filtering, CFN changeset vs direct-create strategy.

---

##### 4. Automatic Rollback (Transaction-like Semantics)

**Principle:** If any stack fails, undo everything deployed in this run without touching pre-existing infrastructure.

```bash
# Pre-run: snapshot what already exists
cp "$OUT_STACKS_FILE" "$PRE_RUN_STACKS_FILE" 2>/dev/null || touch "$PRE_RUN_STACKS_FILE"
> "$OUT_STACKS_FILE"  # Fresh tracking for this run

# ERR trap:
trap rollback ERR

rollback() {
  local EXIT_CODE=$?
  
  # Load stacks deployed THIS run
  DEPLOYED_STACKS=()
  while IFS= read -r line; do DEPLOYED_STACKS+=("$line"); done < "$OUT_STACKS_FILE"
  
  # Load pre-existing stacks (DON'T delete these)
  PRE_RUN_STACKS=()
  while IFS= read -r line; do PRE_RUN_STACKS+=("$line"); done < "$PRE_RUN_STACKS_FILE"
  
  # Handle IAM deny-group blocking deletes
  IN_GROUP=$(aws iam list-groups-for-user ...)
  [[ -n "$IN_GROUP" ]] && aws iam remove-user-from-group ...
  
  # Delete in REVERSE order (respects dependencies)
  for (( i=${#DEPLOYED_STACKS[@]}-1; i>=0; i-- )); do
    STACK="${DEPLOYED_STACKS[$i]}"
    # Skip if it existed before this run
    printf '%s\n' "${PRE_RUN_STACKS[@]}" | grep -qx "$STACK" && continue
    aws cloudformation delete-stack --stack-name "$STACK"
    aws cloudformation wait stack-delete-complete --stack-name "$STACK"
  done
  
  # Cleanup
  > "$OUT_STACKS_FILE"
  # Re-add user to deny-group
  aws iam add-user-to-group ...
  exit "$EXIT_CODE"
}
```

**Skills demonstrated:** Transaction semantics in shell, array manipulation, set operations (pre-run vs current-run), reverse iteration, IAM management, graceful cleanup.

---

##### 5. Parallel Execution (Concurrency in Bash)

**Principle:** Independent stacks should deploy simultaneously to reduce total time.

```bash
_PARALLEL_PIDS=()
_PARALLEL_LOGS=()
_PARALLEL_NAMES=()

deploy_stack_parallel() {
  local STACK_NAME="$1" TEMPLATE="$2" PARAMS="${3:-}"
  local PLOG=$(mktemp)  # Temp log per parallel job
  _PARALLEL_LOGS+=("$PLOG")
  _PARALLEL_NAMES+=("$STACK_NAME")
  
  # Run in background subshell — disable ERR trap inside child
  ( trap - ERR; deploy_stack "$STACK_NAME" "$TEMPLATE" "$PARAMS" >"$PLOG" 2>&1 ) &
  _PARALLEL_PIDS+=("$!")
}

wait_parallel() {
  local ALL_OK=true
  for i in "${!_PARALLEL_PIDS[@]}"; do
    if wait "${_PARALLEL_PIDS[$i]}"; then
      echo "  ✔️  [parallel] ${_PARALLEL_NAMES[$i]}"
    else
      echo "  ❌ [parallel] ${_PARALLEL_NAMES[$i]} FAILED"
      ALL_OK=false
    fi
    cat "${_PARALLEL_LOGS[$i]}"  # Print captured output
    rm -f "${_PARALLEL_LOGS[$i]}"
  done
  # Reset for next group
  _PARALLEL_PIDS=(); _PARALLEL_LOGS=(); _PARALLEL_NAMES=()
  [[ "$ALL_OK" == "false" ]] && return 1
}
```

**Skills demonstrated:** Process management, background jobs (`&`), PID tracking, `wait` with exit code capture, temp file per-process isolation, array management for multiple parallel groups.

---

##### 6. Cross-Stack Output Passing

**Principle:** Downstream stacks need values from upstream stacks (VPC ID, API Gateway ID, Secret ARN).

```bash
get_output() {
  local STACK_NAME="$1" OUTPUT_KEY="$2"
  aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" --region "$REGION" \
    --query "Stacks[0].Outputs[?OutputKey=='${OUTPUT_KEY}'].OutputValue" \
    --output text
}

# After Redshift stack deploys, read its secret ARN:
REDSHIFT_SECRET=$(get_output "${USE_CASE}-redshift-serverless" "RedshiftSecretArn")

# Inject into parms.json for all downstream stacks:
python3 -c "
import json
with open('$PARMS') as f: d = json.load(f)
d['Parameters']['RedshiftSecretArn'] = '$REDSHIFT_SECRET'
with open('$PARMS', 'w') as f: json.dump(d, f)
"
```

**Skills demonstrated:** CFN output consumption, dynamic parameter file mutation, JSON manipulation with Python, dependency chain management.

---

##### 7. Auto-Discovery (Runtime Intelligence)

**Principle:** Don't hardcode IDs that can change. Discover them from live AWS APIs.

```bash
# Discover AI agent runtime ID from Bedrock AgentCore:
RUNTIME_ID=$(aws bedrock-agentcore-control list-agent-runtimes \
  --region "$REGION" --output json | python3 -c "
import sys, json
data = json.load(sys.stdin)
for r in data.get('agentRuntimes', []):
    if 'tor' in r['agentRuntimeName'].lower():
        print(r['agentRuntimeId'])
        break
")

# Discover Cognito client secret (not CFN-exportable):
SECRET=$(aws cognito-idp describe-user-pool-client \
  --user-pool-id "$POOL_ID" --client-id "$CLIENT_ID" \
  --query "UserPoolClient.ClientSecret" --output text)

# Discover CloudFront distribution ID from domain:
CF_DIST_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?DomainName=='${CF_DOMAIN}'].Id | [0]" --output text)
```

**Skills demonstrated:** AWS API discovery patterns, JMESPath queries, Python for complex response parsing, graceful fallbacks when discovery fails.

---

##### 8. Concurrent-Run Safety

**Principle:** Two engineers (or CI jobs) running the script simultaneously must not corrupt each other.

```bash
# Problem: Shared /tmp/parms.json would cause race conditions
# Solution: PID-scoped temp files
PARMS_TMP=$(mktemp "/tmp/parms_${$}_XXXXXX.json")  # $$ = current PID
chmod 600 "$PARMS_TMP"  # Restrict before writing secrets

# Cleanup on exit (covers normal exit, errors, and signals):
trap 'rm -f "${PARMS_TMP:-}" "${OTHER_TEMPS[@]:-}"' EXIT
```

**Skills demonstrated:** Race condition awareness, `mktemp` for safe temp files, PID isolation, permission hardening (CWE-732), comprehensive trap cleanup.

---

##### 9. Observability & Audit Trail

**Principle:** Every deploy must be fully traceable for debugging and compliance.

```bash
# Tee ALL output (stdout + stderr) to timestamped log:
LOG_FILE="deploy-$(date +%Y%m%d-%H%M%S).log"
exec > >(tee -a "$LOG_FILE") 2>&1

# Per-stack timing:
STACK_START=$(date +%s)
# ... deploy ...
STACK_ELAPSED=$(( $(date +%s) - STACK_START ))
echo "  ✔️  Deployed: $STACK_NAME (${STACK_ELAPSED}s)"

# Total elapsed:
TOTAL_ELAPSED=$(( $(date +%s) - SCRIPT_START ))
echo "Total: $(( TOTAL_ELAPSED / 60 ))m $(( TOTAL_ELAPSED % 60 ))s"

# Stack tracking file (for rollback + teardown):
echo "$STACK_NAME" >> "$OUT_STACKS_FILE"
```

**Skills demonstrated:** Process substitution (`>(tee ...)`), file descriptor redirection, arithmetic timing, audit file generation.

---

#### Design Decisions & Trade-offs

| Decision | Why |
|----------|-----|
| **Bash over Python/Terraform** | Zero additional dependencies — every AWS environment has bash + AWS CLI. No pip/terraform install needed. |
| **Single file (3200 lines)** | One `scp` or `git clone` and you can deploy. No module resolution, no imports to break. |
| **Inline Python for parsing** | Bash can't parse JSON/YAML reliably. Python3 is always present. Keeps logic co-located. |
| **create-stack vs deploy (changeset)** | New stacks use `create-stack` to bypass EarlyValidation hooks. Existing stacks use `deploy` for safe updates. |
| **Parallel groups (not full DAG)** | Full DAG scheduling in bash is complex. Grouping independent stacks by phase is simple and covers 80% of parallelism benefit. |
| **Skip-on-success (not force-redeploy)** | Most re-runs are after a partial failure. Skipping completed stacks saves 15+ minutes on retry. |
| **Pre-run snapshot for rollback** | Prevents catastrophic deletion of pre-existing infrastructure on a failed update run. |
| **Selective filter (--stack)** | For single-stack changes, 90% of re-runs only need one stack. Full orchestration is overkill. |

---

### Summary: What This Script Demonstrates

If asked "what's your most complex piece of work" in an interview, this script demonstrates:

1. **Systems thinking** — understanding 40+ service dependencies and ordering them correctly
2. **Resilience engineering** — retry, backoff, rollback, idempotency, failure classification
3. **Automation at scale** — one command provisions an entire production environment
4. **AWS depth** — 20+ services called programmatically, deep CFN lifecycle knowledge
5. **Bash mastery** — traps, subshells, arrays, process management, IPC via temp files
6. **Security awareness** — temp file permissions, deny-group handling, secret management
7. **Developer experience** — clear progress output, selective re-runs, dry-run mode, help text
8. **Operational maturity** — audit logs, teardown script, validation script, safe re-runs
# Stacks tracked in all-stacks.txt for rollback/teardown
```

---

### Teardown Script

Cleanly removes all deployed resources respecting dependency order:

```bash
# 1. Empty S3 buckets (CFN cannot delete non-empty buckets)
# 2. Delete all CFN stacks in REVERSE order
# 3. Wait for each deletion
# 4. Clean orphaned non-CFN resources:
#    - SSM parameters
#    - Imperatively-created S3 buckets
#    - Orphaned Secrets Manager entries (force-delete)
```

---

## 4. Containerization & CI/CD

### Docker Image Strategy

| Component | Base Image | Platform | Purpose |
|-----------|-----------|----------|---------|
| AI Agent (TOR) | `ghcr.io/astral-sh/uv:python3.14-bookworm-slim` | linux/arm64 | AgentCore runtime container |
| AI Agent (Chatbot) | `ghcr.io/astral-sh/uv:python3.12-bookworm-slim` | linux/arm64 | AgentCore runtime container |
| UI Build | `node:22` + AWS CLI v2 | linux/amd64 | Ephemeral build container |

### Agent Container Deployment Pipeline

```
Source ZIP → Repack (Dockerfile at root) → S3 Upload
    → CodeBuild (ARM64 Docker build) → ECR Push
    → AgentCore Runtime Update → Poll until READY
    → Update proxy Lambda env vars (runtime ARN)
```

**Key implementation details:**

```bash
# 1. Create ECR repo if not exists
aws ecr create-repository --repository-name <repo>

# 2. Set ECR resource policy for CodeBuild push access
aws ecr set-repository-policy --policy-text '{"Statement":[...]}'

# 3. Repack ZIP — flatten subdirectories, ensure Dockerfile at root
# (Python script handles zip structure normalization)

# 4. Generate buildspec.yml dynamically (inline in the source zip)
# Buildspec: ecr login → docker buildx → tag → push

# 5. Upload source to S3 for CodeBuild
aws s3 cp source.zip s3://codebuild-sources-bucket/agent/source.zip

# 6. Create/update CodeBuild project (ARM64, inline buildspec)
aws codebuild create-project --environment type=ARM_CONTAINER,image=aws/codebuild/amazonlinux2-aarch64-standard:3.0

# 7. Start build and poll for completion
BUILD_ID=$(aws codebuild start-build --project-name <project> --query 'build.id')
# Poll every 15s until SUCCEEDED/FAILED

# 8. Update AgentCore runtime with new image URI
python3 -c "
boto3.client('bedrock-agentcore-control').update_agent_runtime(
    agentRuntimeId='<id>',
    agentRuntimeArtifact={'containerConfiguration': {
        'containerUri': '<account>.dkr.ecr.<region>.amazonaws.com/<repo>:latest'
    }}
)"

# 9. Poll runtime status until READY (12s intervals)
```

### Containerized UI Build

```bash
# Build Docker image (Node 22 + AWS CLI)
docker build --platform linux/amd64 -t ui-builder .

# Run deployment inside container (mounts repo + AWS creds)
docker run --rm \
  -v "${REPO_ROOT}:/app" \
  -v "${HOME}/.aws:/root/.aws:ro" \
  ui-builder bash /app/deploy-ui.sh

# Inside container:
# 1. Read all config from SSM Parameter Store
# 2. Generate .env file for Vite build
# 3. npm install && npm run build
# 4. aws s3 sync dist/ s3://<bucket>/ (with cache headers)
# 5. aws cloudfront create-invalidation --paths "/*"
```

**Why Docker for UI build?**
- No local Node.js version conflicts
- Reproducible builds across developer machines
- AWS CLI version consistency
- Clean environment every time (no leftover node_modules)

---

## 5. AWS Services Expertise

### Services I Deploy & Manage

| Category | Service | What I Do |
|----------|---------|-----------|
| **Compute** | Lambda (22 functions) | Deploy via CFN, manage code on S3, set VPC config, layers |
| | Bedrock AgentCore | Build ARM64 containers, deploy runtimes, manage lifecycle |
| | CodeBuild | Configure projects, manage buildspecs, ARM64 builds |
| | SageMaker Endpoint | Deploy ML model endpoints via CFN |
| | Glue ETL | Deploy jobs + connections, manage scripts on S3 |
| **Networking** | VPC + Subnets | Provision with DNS support enabled post-create |
| | API Gateway (REST) | 50+ routes, Lambda integrations, CORS, OPTIONS methods |
| | API Gateway (HTTP) | Lightweight API for specific endpoints |
| | API Gateway (WebSocket) | Real-time push notifications |
| | CloudFront | CDN with OAC, custom error pages for SPA routing |
| **Storage** | S3 (5+ buckets) | Lambda code, static site, logs, reports, pipeline data |
| | ECR | Docker image repositories for AI agents |
| **Database** | Redshift Serverless | Schema DDL, materialized views, scheduled refresh |
| | DynamoDB (10+ tables) | GSIs, PAY_PER_REQUEST billing, data seeding |
| | Timestream | IoT time-series ingestion via IoT Core rules |
| **Auth** | Cognito | User pools, app clients, OAuth2 flows, callback URLs |
| | Secrets Manager | Redshift credentials lifecycle |
| **Messaging** | SNS | Alert notification topics |
| | EventBridge | Scheduling, event routing, rule-based triggers |
| | Firehose | Streaming data to Redshift |
| | IoT Core | Sensor data ingestion rules |
| **Management** | SSM Parameter Store | 30+ params, runtime config, cross-service discovery |
| | CloudWatch | Log groups, retention policies, metric monitoring |
| | IAM | Roles for Lambda, AgentCore, CodeBuild, SageMaker, Glue |

---

## 6. Database Management

### Redshift Serverless

**What I manage:**
- Schema DDL (10 base tables, 9 materialized views)
- Workgroup + namespace provisioning via CFN
- Secrets Manager credential management
- Materialized view refresh scheduling (Lambda + EventBridge)
- Data API access from Lambda functions

**Materialized Views pattern:**
```sql
-- MVs aggregate operational data for fast API queries
-- Partitioned by ROW_NUMBER() for "latest record per entity"
-- Filtered by is_valid = true AND is_deleted = false (soft deletes)
-- 90-day and 180-day rolling windows for trending
-- Scheduled refresh via Lambda + EventBridge
CREATE MATERIALIZED VIEW mv_latest AS
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY entity_id ORDER BY timestamp DESC) AS rn
    FROM base_table WHERE is_valid = true AND is_deleted = false
)
SELECT * FROM ranked WHERE rn = 1;
```

### DynamoDB

**What I manage:**
- Table provisioning via CFN (PAY_PER_REQUEST billing)
- Global Secondary Index design for query access patterns
- Data seeding scripts (Python/boto3)
- Table-per-concern design (alerts, orders, jobs, connections, state, audit)

**Access patterns:**
```python
# GSI design for multi-dimensional queries:
# alert_id (PK) + GSIs on: trial_id, site_id, severity+status
# Enables: get by ID, filter by entity, filter by severity
```

---

## 7. Configuration Management

### SSM Parameter Store Strategy

**Why SSM:**
- Single source of truth for all runtime configuration
- Decouples infrastructure outputs from application code
- Cross-service discovery without hardcoding
- SecureString type for secrets
- Applications resolve config at startup (Lambda env vars or runtime reads)

**Parameter Hierarchy:**
```
/<use-case>/
├── Core (use-case-name, region, account-id)
├── Database (redshift-workgroup, redshift-database, redshift-secret-arn)
├── DynamoDB (table names for all 10+ tables)
├── API Endpoints (rest-api-endpoint, http-api-url, websocket-endpoint)
├── Frontend (cloudfront-domain, distribution-id, ui-s3-bucket)
├── Auth (cognito-user-pool-id, client-id, domain-url, client-secret [SecureString])
├── AI Agents (runtime-ids, invocation-urls)
├── ML (sagemaker-feature-group, endpoint-name)
└── Infrastructure (lambda-code-bucket)
```

**How I seed SSM:**
```bash
# Read CFN stack outputs → write to SSM
REST_API_ID=$(aws cloudformation describe-stacks \
  --stack-name "${USE_CASE}-rest-api-gateway" \
  --query "Stacks[0].Outputs[?OutputKey=='RestApiId'].OutputValue" --output text)

aws ssm put-parameter \
  --name "/${USE_CASE}/rest-api-endpoint" \
  --value "https://${REST_API_ID}.execute-api.${REGION}.amazonaws.com/prod" \
  --type String --overwrite

# Cognito client secret (not CFN-exportable) → SecureString
COGNITO_SECRET=$(aws cognito-idp describe-user-pool-client \
  --user-pool-id "$POOL_ID" --client-id "$CLIENT_ID" \
  --query "UserPoolClient.ClientSecret" --output text)

aws ssm put-parameter \
  --name "/${USE_CASE}/cognito-app-client-secret" \
  --value "$COGNITO_SECRET" --type SecureString --overwrite
```

**Cross-region deployment:**
- Seed placeholder SSM values in target region before deploying stacks
- CFN templates that resolve `ssm:` dynamic references won't fail on missing keys
- Real values overwrite placeholders after their respective stacks deploy

---

## 8. Security & IAM

### IAM Role Management

| Role | Used By | Permissions |
|------|---------|-------------|
| Lambda Execution Roles | Each Lambda function | DynamoDB, Redshift Data API, S3, SSM, CloudWatch Logs |
| AgentCore Runtime Role | AI agent containers | Redshift, Secrets Manager, S3, Bedrock |
| CodeBuild Service Role | Docker image builds | ECR push/pull, S3 source bucket, CloudWatch Logs |
| SageMaker Execution Role | ML Endpoint + Feature Store | S3, Glue, Feature Store, Model hosting |
| Glue Service Role | ETL jobs | Redshift connection, S3, Feature Store |

### Authentication Architecture

```
User → CloudFront → Cognito Hosted UI → OAuth2 Code Flow → JWT
JWT → API Gateway (pass-through) → Lambda
Lambda → AgentCore (SigV4 — Lambda role signs the request)
```

**Key points:**
- AI agent runtimes use SigV4 only (no JWT on runtime itself)
- Proxy Lambda signs requests with its IAM execution role
- Cognito app client callback URLs updated programmatically after CloudFront deploys

### Deny-Group Handling (IAM Group with Explicit Deny)

```bash
# Problem: IAM group has explicit Deny on Redshift that blocks stack operations
# Solution: Transparently remove user before ops, re-add after

CURRENT_USER=$(aws sts get-caller-identity --query 'Arn' | awk -F/ '{print $NF}')
IN_GROUP=$(aws iam list-groups-for-user --user-name "$CURRENT_USER" \
  --query "Groups[?GroupName=='deny-group'].GroupName" --output text)

if [[ -n "$IN_GROUP" ]]; then
  aws iam remove-user-from-group --user-name "$CURRENT_USER" --group-name "$DENY_GROUP"
  # ... do Redshift operations ...
  aws iam add-user-to-group --user-name "$CURRENT_USER" --group-name "$DENY_GROUP"
fi
```

### Secret Management

- Redshift credentials stored in Secrets Manager (auto-created by CFN)
- Cognito client secret stored as SSM SecureString
- Lambda env vars reference SSM paths (not raw secrets)
- Docker containers receive config via AgentCore runtime environment variables

---

## 9. Monitoring, Logging & Troubleshooting

### Log Management

```bash
# Set 30-day retention on all Lambda log groups (default is infinite):
for LOG_GROUP in $(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/${USE_CASE}" \
  --query 'logGroups[*].logGroupName' --output text); do
  aws logs put-retention-policy --log-group-name "$LOG_GROUP" --retention-in-days 30
done
```

### Common Failure Scenarios & Resolution

| Issue | Root Cause | Resolution |
|-------|-----------|------------|
| Stack stuck in `ROLLBACK_COMPLETE` | Failed creation | Auto-delete tombstone + retry fresh |
| Stack in `REVIEW_IN_PROGRESS` | Stale changeset from aborted run | Delete changeset + retry |
| `EarlyValidation` failure | Resource exists outside CFN | Delete tombstone → CFN adopts existing resource |
| CodeBuild `sts:AssumeRole` error | Wrong role ARN in buildspec | Update CodeBuild project service role |
| AgentCore runtime not `READY` | Container crash (missing module) | Check CloudWatch for `ImportError`/`Traceback` |
| Lambda timeout in VPC | DNS resolution failure | `modify-vpc-attribute --enable-dns-support true` |
| CFN deploy "No updates are to be performed" | Template unchanged | `--no-fail-on-empty-changeset` flag handles this |
| S3 bucket deletion fails | Bucket not empty | Empty bucket first with `aws s3 rm --recursive` |
| Cognito hosted UI `/error` page | Missing identity provider on app client | `update-user-pool-client --supported-identity-providers COGNITO` |
| Cross-region deploy fails on SSM reference | SSM key doesn't exist in target region | Pre-seed placeholders before stack deploy |

### Health Check Commands

```bash
# All stack statuses
aws cloudformation list-stacks --region us-west-2 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[?starts_with(StackName,`prefix`)].{Name:StackName,Status:StackStatus}'

# AI agent runtime status
python3 -c "
import boto3
c = boto3.client('bedrock-agentcore-control', region_name='us-west-2')
for r in c.list_agent_runtimes()['agentRuntimes']:
    print(f\"{r['agentRuntimeName']}: {r['status']}\")
"

# Agent container logs (last 10 min)
aws logs tail /aws/bedrock-agentcore/runtimes/<RUNTIME_ID>-DEFAULT --since 10m

# Lambda errors (last 1 hour)
aws logs filter-log-events \
  --log-group-name /aws/lambda/<function-name> \
  --start-time $(date -v-1H +%s000) --filter-pattern "ERROR"

# Redshift connectivity test
aws redshift-data execute-statement \
  --workgroup-name <workgroup> --database <db> --sql "SELECT 1"
```

---

## 10. Shell Scripting Patterns & Techniques

### Error Handling & Traps
```bash
set -euo pipefail          # Exit on error, unset vars, pipe failures
trap rollback ERR          # Auto-rollback on any error
trap 'rm -f $TEMP_FILES' EXIT  # Cleanup temp files on exit
```

### Safe Temp File Management
```bash
# Concurrent-run safe — no shared /tmp paths
PARMS_TMP=$(mktemp "/tmp/parms_${$}_XXXXXX.json")
chmod 600 "$PARMS_TMP"    # Restrict before secrets are written
```

### Argument Parsing
```bash
while [[ $# -gt 0 ]]; do
  case "$1" in
    --region)     REGION="$2"; shift 2 ;;
    --dry-run)    DRY_RUN=true; shift ;;
    --stack)      STACK_FILTER="$2"; shift 2 ;;
    --help|-h)    usage; exit 0 ;;
    *)            echo "❌ Unknown: $1"; usage; exit 1 ;;
  esac
done
```

### Inline Python for Complex Logic
```bash
# Parse JSON/YAML, filter parameters, process API responses:
RESULT=$(python3 -c "
import json, sys
data = json.load(sys.stdin)
# ... complex filtering logic ...
print(result)
" <<< "$JSON_INPUT")
```

### Polling with Timeout
```bash
local _poll_secs=0
while [[ $_poll_secs -lt 900 ]]; do   # 15 min timeout
  STATUS=$(aws cloudformation describe-stacks ...)
  [[ "$STATUS" == "COMPLETE" ]] && break
  [[ "$STATUS" == "FAILED" ]] && { handle_failure; break; }
  echo "  ⏳ ${STATUS} — waited ${_poll_secs}s..."
  sleep 30
  _poll_secs=$(( _poll_secs + 30 ))
done
```

### S3 Upload with Retry
```bash
_S3_UPLOAD_OK=false
for _retry in 1 2 3; do
  if aws s3 cp "$file" "s3://${BUCKET}/${key}"; then
    _S3_UPLOAD_OK=true; break
  fi
  echo "  ⚠️  Attempt $_retry failed — retrying in 5s..."
  sleep 5
done
[[ "$_S3_UPLOAD_OK" == "false" ]] && { echo "❌ Failed after 3 attempts"; exit 1; }
```

### Dynamic Parameter File Generation
```bash
# Generate JSON with all resource names derived from USE_CASE:
cat > "$PARMS" <<EOF
{
  "Parameters": {
    "UseCaseName": "${USE_CASE}",
    "LambdaCodeBucket": "${LAMBDA_CODE_BUCKET}",
    "RedshiftWorkgroup": "${USE_CASE}-workgroup",
    "RedshiftDatabase": "${USE_CASE}_db",
    "FeatureGroupName": "${USE_CASE//-/_}_demand_features_v2",
    "SklearnContainerImage": "${SKLEARN_ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/sagemaker-scikit-learn:1.4-2-cpu-py3"
  }
}
EOF
```

### Region-Specific Logic (Bash 3.2 compatible — no associative arrays)
```bash
case "${REGION}" in
  us-east-1)      SKLEARN_ACCOUNT="683313688378" ;;
  us-west-2)      SKLEARN_ACCOUNT="246618743249" ;;
  eu-west-1)      SKLEARN_ACCOUNT="141502667606" ;;
  *)              echo "ERROR: Unsupported region"; exit 1 ;;
esac
```

### Log Tee + Progress Timers
```bash
LOG_FILE="deploy-$(date +%Y%m%d-%H%M%S).log"
exec > >(tee -a "$LOG_FILE") 2>&1       # All output to terminal + file
SCRIPT_START=$(date +%s)                 # Total elapsed timer
# ... per-stack timing ...
STACK_ELAPSED=$(( $(date +%s) - STACK_START ))
echo "  ✔️  Deployed: $STACK_NAME (${STACK_ELAPSED}s)"
```

---

## 11. Key Commands Cheat Sheet

### Deployment
```bash
# Full environment deploy
./0-oneclick-setup.sh --numpy-layer-arn <ARN> --region us-west-2

# Selective stack redeploy
./0-oneclick-setup.sh --numpy-layer-arn <ARN> --stack <partial-name>

# Dry run (validate only)
./0-oneclick-setup.sh --numpy-layer-arn <ARN> --dry-run

# UI only
cd UI && ./docker-build-deploy.sh <use-case> <region>

# Agent redeploy
agentcore deploy --agent <agent-name> --auto-update-on-conflict
```

### Teardown
```bash
./scripts/99-destroy.sh --use-case <name> --region <region>
```

### Configuration
```bash
# Seed all SSM parameters
./scripts/parameter_store.sh <use-case> <region> <account-id>

# Read a parameter
aws ssm get-parameter --name "/<use-case>/rest-api-endpoint" --query Parameter.Value --output text

# Update a parameter
aws ssm put-parameter --name "/<use-case>/key" --value "new-value" --overwrite
```

### Database
```bash
# Run Redshift DDL
aws redshift-data execute-statement \
  --workgroup-name <workgroup> --database <db> \
  --sql "$(cat sql/redshift_ddl.sql)"

# Refresh materialized views
aws redshift-data execute-statement \
  --workgroup-name <workgroup> --database <db> \
  --sql "REFRESH MATERIALIZED VIEW mv_name;"

# Seed DynamoDB
python3 scripts/seed_all_tables.py
```

### Monitoring
```bash
# CloudFront invalidation
aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"

# Agent health
python3 -c "
import boto3
c = boto3.client('bedrock-agentcore-control', region_name='<region>')
for r in c.list_agent_runtimes()['agentRuntimes']:
    print(r['agentRuntimeName'], r['status'])"

# Lambda logs
aws logs tail /aws/lambda/<function> --since 10m --filter-pattern "ERROR"

# CodeBuild status
aws codebuild batch-get-builds --ids <build-id> --query 'builds[0].buildStatus'
```

### Validation
```bash
# Validate all CFN templates
./scripts/validate_cfn.sh --region <region>

# Validate single template
aws cloudformation validate-template --template-body file://template.yaml
```

---

## 12. Interview Q&A

### "What are your day-to-day DevOps responsibilities?"

> I own the full infrastructure lifecycle for a multi-tier AWS application — 40+ CloudFormation stacks spanning compute, networking, databases, ML, and frontend. Day-to-day I write CFN templates, maintain a 3200-line deployment orchestrator with retry/rollback/parallel execution, deploy containerized AI agents to Bedrock AgentCore via CodeBuild/ECR, manage Redshift schemas, handle SSM configuration, build Docker-based CI pipelines, and troubleshoot deployment failures. I also manage the frontend deployment pipeline (containerized Node.js build → S3 → CloudFront invalidation).

### "Tell me about a complex automation you built from scratch"

> I designed and built a 3200-line Bash orchestrator that deploys 40+ interdependent CloudFormation stacks in one command. The core challenge was ordering dependencies correctly — some stacks need outputs from others (VPC IDs, API IDs, secret ARNs). I built a state machine that checks each stack's current status before deciding what to do (skip, create, delete tombstone + recreate, or wait). I added retry with exponential backoff for transient failures, automatic rollback via ERR trap that deletes stacks in reverse order (while preserving pre-existing ones), parallel execution for independent stacks using background subshells with per-process log capture, and inline Python for JSON/YAML parsing that Bash can't handle. The script is fully idempotent — safe to re-run after any failure — and uses mktemp for concurrent-run safety so two engineers can deploy simultaneously without corrupting each other's state.

### "How do you handle infrastructure deployments?"

> I built a one-click orchestration script that deploys everything in dependency order. Key features:
> - Pre-flight validation (tools, credentials, artifacts — fail fast)
> - Idempotent re-runs — observes stack state and skips already-deployed stacks
> - Retry with exponential backoff (3 attempts, 15s/30s backoff)
> - Failure classification — retries transient errors, stops on terminal failures
> - Automatic rollback on failure (reverse-order deletion, preserving pre-existing stacks)
> - Parallel deployment for independent stacks (background subshells + PID tracking)
> - Cross-stack output passing (read CFN outputs → inject into downstream parameters)
> - Selective redeployment via `--stack` filter
> - Cross-region support with auto-derived resource names
> - Timestamped audit logs + per-stack elapsed timers

### "How do you handle failures and rollback?"

> The script uses an ERR trap for automatic rollback. It tracks stacks deployed in the current run (separate from pre-existing stacks), then deletes them in reverse dependency order. It handles edge cases like deny-group membership blocking deletions, stale changesets, and tombstone stacks. Each individual stack also has retry logic — it detects transient failures vs terminal failures and only retries when recovery is possible.

### "Describe your CI/CD pipeline"

> For AI agents: source ZIP → S3 upload → CodeBuild (ARM64 Docker build) → ECR push → AgentCore runtime update → poll until READY → update proxy Lambda env vars. For the UI: Docker container builds React app (no local Node.js dependency) → S3 sync with appropriate cache headers → CloudFront invalidation. For infrastructure: CFN templates validated → deployed in dependency order with the orchestrator handling the full lifecycle.

### "How do you manage configuration across environments?"

> All runtime configuration flows through SSM Parameter Store (30+ parameters). After each deploy, a seeding script reads CFN stack outputs and writes them to SSM. Applications resolve config from SSM at startup. For cross-region deploys, I pre-seed placeholder values so CFN dynamic references don't fail. Secrets use SSM SecureString or Secrets Manager.

### "What's your approach to infrastructure-as-code?"

> Everything is CloudFormation YAML organized into 8 logical domains. Templates are parameterized for multi-environment reuse. I use conditions for optional resources, DeletionPolicy:Retain for stateful resources, and cross-stack output passing for dependency management. I validate templates both via the CFN API and with static analysis for hardcoded references. Large templates (>51KB) are handled via S3 upload.

### "How do you handle secrets and security?"

> Authentication uses Cognito (OAuth2 code flow). AI agents use SigV4 signing (proxy Lambda signs with its IAM role). Secrets live in Secrets Manager (DB creds) or SSM SecureString (app secrets). Each service has a dedicated least-privilege IAM role. The deployment script handles deny-group membership transparently when operations require elevated Redshift access.

### "How do you monitor and troubleshoot?"

> CloudWatch is primary — all Lambda log groups have 30-day retention set automatically. AgentCore runtimes write to dedicated log groups. I check agent health via the AgentCore API (list runtimes → check status). For deployment failures, the orchestrator generates timestamped log files. Common issues (tombstone stacks, stale changesets, EarlyValidation) are handled automatically. For app-level issues, I filter CloudWatch logs for ERROR patterns and check Redshift connectivity.

### "Tell me about your containerization work"

> I deploy ARM64 Docker containers to Bedrock AgentCore. The pipeline: normalize source ZIP structure → generate buildspec → upload to S3 → CodeBuild builds multi-stage Docker image → pushes to ECR → update AgentCore runtime → poll until READY. I also use Docker for the frontend build to eliminate local toolchain dependencies — the build runs inside a Node 22 container that's purpose-built for the job.

### "How do you handle multi-region deployments?"

> The deployment script accepts a `--region` flag and auto-derives resource names (S3 bucket suffixes, ECR URIs). For cross-region, I pre-seed SSM placeholder values so CFN templates with dynamic references don't fail. SageMaker container image URIs are region-specific (I maintain a lookup by region). The same templates deploy to any supported region without modification.

### "What's your experience with databases?"

> I manage Redshift Serverless (DDL schema, 9 materialized views with scheduled refresh via Lambda + EventBridge), DynamoDB (10+ tables with GSI design for access patterns, PAY_PER_REQUEST billing), and Timestream (IoT time-series via IoT Core rules). I write data seeding scripts in Python/boto3 and manage the Glue ETL pipeline that transforms Redshift data into SageMaker Feature Store features.

### "How long does deployment take?"

> Full environment from scratch: 18-25 minutes. Bottlenecks are Redshift Serverless creation (5+ min) and CodeBuild for agent images (5-8 min each). Single stack updates: 30-90 seconds. UI-only redeploy: ~3 minutes. Selective stack redeploy: under 2 minutes.
