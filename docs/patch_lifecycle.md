# Patch Lifecycle Guide

## Overview
This document defines the complete lifecycle of Atlas patches: from generation through verification, refinement, application, and potential rollback. Every stage includes safety controls and multi-layer audit trails.

## Patch Verification

### Worktree-Based Isolation
Atlas verifies patches in temporary git worktrees to avoid polluting the main branch.

**Process**:
1. Create isolated worktree: `git worktree add atlas-verify-<timestamp> HEAD`
2. Apply patch: `git apply <patch_diff>`
3. Run verification steps (see below)
4. Clean up: `git worktree remove atlas-verify-<timestamp>`

**Benefits**:
- Main branch remains untouched during testing
- Multiple patches can be verified in parallel
- Rollback is automatic if verification fails (just delete worktree)

### Verification Steps

#### 1. Build Verification
Run the target repository's canonical build command.

**Example** (Windows ROCm repo):
```powershell
cd atlas-verify-12345
powershell -ExecutionPolicy Bypass -File .\build.ps1
```

**Success Criteria**: Exit code 0, no build errors in output

#### 2. Test Suite Execution
Run `test_commands` from the LLM patch response.

**Configuration** (`llm_config.yaml`):
```yaml
target_repos:
  ROCm_Installer:
    build_command: "powershell -ExecutionPolicy Bypass -File .\\build.ps1"
    test_commands:
      - "pytest tests/"
      - "powershell .\\validate.ps1"
```

**Success Criteria**: All test commands exit with code 0

#### 3. Result Aggregation
```json
{
  "patch_id": "atlas-patch-20251029-143022",
  "build_status": "pass",
  "test_results": [
    {"command": "pytest tests/", "exit_code": 0, "duration_seconds": 12.4},
    {"command": "powershell .\\validate.ps1", "exit_code": 0, "duration_seconds": 5.1}
  ],
  "verification_status": "pass"
}
```

### Verification Failures

**On Build Failure**:
- Capture full stderr and compiler diagnostics
- Trigger refinement cycle with build context
- After max retries (default: 3), escalate to operator

**On Test Failure**:
- Capture failing test names and assertion errors
- Include in refinement prompt to focus LLM attention
- Present test diff if available (expected vs. actual)

## Rollback Mechanisms

### Rollback Triggers

#### 1. Automatic Rollback on CI Failure (Manual Confirmation Required)
**Scenario**: Patch is merged, but subsequent CI run detects failure

**Process**:
1. CI workflow detects failure in post-merge validation
2. Atlas creates rollback proposal with failure context
3. Operator notified via Streamlit UI and/or GitHub comment
4. Operator reviews failure logs and confirms rollback
5. Atlas executes `git revert <patch_commit>` and pushes to `Master`

**Configuration Flag**:
```yaml
rollback:
  auto_detect_ci_failure: true
  require_manual_confirmation: true  # Never auto-revert without approval
```

#### 2. Manual Operator-Triggered Rollback
**Scenario**: Operator identifies issue not caught by CI (e.g., performance regression)

**Process**:
1. Operator navigates to Rollback tab in Streamlit UI
2. Views recent Atlas commits with metadata
3. Selects commit to revert
4. Reviews generated revert command
5. Confirms and executes rollback

**UI Flow** (text-based, see `docs/ui_workflows.md`):
- List shows: commit hash, timestamp, confidence score, affected files
- Operator clicks "Prepare Rollback"
- UI displays: `git revert <commit> --no-edit`
- Operator confirms, Atlas executes and logs

#### 3. Time-Based Auto-Revert (Optional, Disabled by Default)
**Scenario**: Stability metrics degrade within monitoring window

**Configuration**:
```yaml
rollback:
  time_based_monitoring:
    enabled: false  # Experimental feature
    window_hours: 24
    stability_threshold: 0.95
    metrics_endpoint: "http://monitoring.internal/api/stability"
```

**Process** (if enabled):
1. Atlas polls metrics endpoint post-merge
2. If stability < threshold within window, propose rollback
3. Operator confirmation still required (never silent revert)

**Recommendation**: Keep disabled until monitoring infrastructure is proven.

## Multi-Layer Audit Trails

Every patch and rollback must be traceable across three layers.

### Layer 1: Git Commit Metadata
**Patch Commits**:
```
atlas: fix build error in module.py

Proposed by: Atlas Agent v0.1.69
Confidence: 0.87
Triggered by: CI failure #142
Verification: All tests passed in worktree atlas-verify-20251029-143022

Affected files:
- src/module.py
- tests/test_module.py

Co-authored-by: Atlas Agent <atlas@gillsystems.local>
```

**Rollback Commits**:
```
Revert "atlas: fix build error in module.py"

This reverts commit abc123def456.

Rollback reason: Post-merge CI detected regression in test_integration.py
Triggered by: Manual operator confirmation (operator: jsmith)
Failure context: TestIntegration::test_startup failed with ImportError

Atlas rollback ID: atlas-rollback-20251029-150315
```

**Benefits**: Immutable history, `git log` provides full context

### Layer 2: JSONL Provenance Log
**Location**: `atlas_core/provenance/patches.jsonl`

**Append-only format** (never modify existing entries):
```jsonl
{"timestamp": "2025-10-29T14:30:22Z", "event": "propose", "patch_id": "atlas-patch-20251029-143022", "confidence_score": 0.87, "llm_endpoint": "http://127.0.0.1:8080", "model": "codellama-13b", "affected_files": ["src/module.py"], "explanation": "Fixed import error"}
{"timestamp": "2025-10-29T14:31:45Z", "event": "verify_pass", "patch_id": "atlas-patch-20251029-143022", "build_status": "pass", "test_results": [{"command": "pytest tests/", "exit_code": 0}]}
{"timestamp": "2025-10-29T14:32:10Z", "event": "apply", "patch_id": "atlas-patch-20251029-143022", "commit_hash": "abc123def456", "operator_confirmed": true, "operator": "jsmith"}
{"timestamp": "2025-10-29T15:03:15Z", "event": "rollback", "patch_id": "atlas-patch-20251029-143022", "rollback_id": "atlas-rollback-20251029-150315", "reason": "ci_failure", "failure_context": "TestIntegration::test_startup failed", "operator_confirmed": true, "operator": "jsmith", "revert_commit": "def456abc789"}
```

**Benefits**: Machine-readable, supports analytics, training data for future LLMs

### Layer 3: GitHub PR/Issue Annotations
**For GitHub-integrated repositories**:

**Patch Application**:
- Comment on triggering issue: "Atlas proposed patch in PR #156"
- Link to patch commit and provenance log entry
- Include confidence score and verification results

**Rollback**:
- Comment on original PR: "⚠️ Patch reverted due to CI failure. See rollback commit def456abc789"
- Link to rollback commit and failure logs
- Tag relevant collaborators

**Configuration**:
```yaml
github_integration:
  enabled: true
  annotate_prs: true
  annotate_issues: true
  tag_on_rollback: ["maintainers"]
```

**Benefits**: Human-readable traceability for team collaboration

## Rollback Procedures

### Standard Rollback Flow
```powershell
# 1. Identify commit to revert
git log --oneline --grep="atlas:" -n 10

# 2. Review commit details
git show <commit_hash>

# 3. Execute revert (Atlas does this after operator confirmation)
git revert <commit_hash> --no-edit

# 4. Verify revert succeeded
git diff HEAD~1 HEAD

# 5. Push to Master (requires ATLAS_PUSH_TOKEN)
git push origin Master
```

### Emergency Rollback (Bypass Atlas)
**When to use**: Atlas agent unavailable, critical production issue

```powershell
# 1. Identify last known good commit
git log --oneline -n 20

# 2. Hard reset (DESTRUCTIVE - use with caution)
git reset --hard <last_good_commit>

# 3. Force push (requires elevated permissions)
git push origin Master --force

# 4. Manual provenance log entry
# Append to atlas_core/provenance/patches.jsonl:
{"timestamp": "2025-10-29T15:30:00Z", "event": "emergency_rollback", "operator": "jsmith", "last_good_commit": "xyz789abc012", "reason": "Critical production issue, Atlas agent offline"}
```

**Warning**: Force pushes bypass all safety controls. Only use in true emergencies.

## Configuration Reference

### Complete `llm_config.yaml` Rollback Section
```yaml
rollback:
  # CI failure detection
  auto_detect_ci_failure: true
  require_manual_confirmation: true  # ALWAYS true in production
  
  # Manual rollback settings
  ui_display_recent_commits: 10
  show_confidence_scores: true
  
  # Time-based monitoring (experimental)
  time_based_monitoring:
    enabled: false
    window_hours: 24
    stability_threshold: 0.95
    metrics_endpoint: ""
  
  # Audit trail
  github_integration:
    enabled: true
    annotate_on_rollback: true
    tag_maintainers: true

verification:
  # Worktree settings
  worktree_prefix: "atlas-verify-"
  cleanup_on_success: true
  cleanup_on_failure: false  # Keep for debugging
  
  # Timeouts
  build_timeout_seconds: 600
  test_timeout_seconds: 300
  
  # Retry logic
  max_retries: 3
  retry_on_transient_errors: true
```

## Best Practices

### For Operators
1. **Always review verification logs** before confirming patch application
2. **Monitor CI for 24 hours** after merging Atlas patches
3. **Use manual rollback** if any unexpected behavior observed, even if CI passes
4. **Never disable `require_manual_confirmation`** in production

### For Developers
1. **Preserve append-only provenance logs** - never edit existing entries
2. **Test rollback procedures** in staging before enabling in production
3. **Mock verification steps** in unit tests to avoid external dependencies
4. **Document custom test commands** in `llm_config.yaml` per target repo

### For Atlas Agent Development
1. **Always create worktrees** for verification - never test in main branch
2. **Capture full context** in rollback commits and logs
3. **Fail safe** - if any verification step is ambiguous, escalate to operator
4. **Log everything** - disk space is cheap, missing audit trails are expensive
