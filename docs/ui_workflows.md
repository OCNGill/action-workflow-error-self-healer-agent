# UI Workflows Guide

## Overview
Atlas provides a **Streamlit-based local web UI** for operator interaction. The UI runs exclusively on `127.0.0.1` and provides manual controls for patch generation, verification, application, and rollback. This guide provides text-based walkthroughs for each workflow.

## Launching the UI

### Prerequisites
```powershell
# Activate conda environment
conda activate atlas

# Verify Streamlit installed
streamlit --version
```

### Start UI
```powershell
# From repo root
cd "c:\Users\Gillsystems Laptop\source\action-workflow-error-self-healer-agent"

# Launch UI (binds to 127.0.0.1 only)
streamlit run atlas_core\ui\streamlit\main.py --server.address 127.0.0.1 --server.port 8501
```

**Access**: Open browser to `http://127.0.0.1:8501`

**Security Note**: UI only accessible from localhost. No remote access by design.

## UI Tab Overview

### Tab Structure
1. **LLM Config** - Configure and test LLM endpoints
2. **Propose** - Generate patches from error context
3. **Verify** - Test patches in isolated worktrees
4. **Apply** - Review and merge verified patches
5. **Rollback** - Revert applied patches with audit trail

## Tab 1: LLM Config

### Purpose
Configure local and cloud LLM endpoints, test connectivity, view current settings.

### Walkthrough

#### Step 1: View Current Configuration
- Navigate to **LLM Config** tab
- Current settings displayed from `atlas_core/config/llm_config.yaml`
- Shows: endpoint URL, model name, enabled status

**Example Display**:
```
Local Endpoint:
  URL: http://127.0.0.1:8080/v1/chat/completions
  Model: codellama-13b
  Status: ✅ Enabled

Cloud Endpoint:
  URL: (Not configured)
  Status: ⚪ Disabled
```

#### Step 2: Test Endpoint Connectivity
- Locate **Test Connection** button under Local Endpoint section
- Click button
- Atlas sends health check request: `{"model": "<model_name>", "messages": [{"role": "user", "content": "ping"}]}`
- Result displayed:

**Success**:
```
✅ Connection successful
Response time: 234ms
Model: codellama-13b
```

**Failure**:
```
❌ Connection failed
Error: Connection refused at http://127.0.0.1:8080
Recommendation: Verify LLM server is running (see docs/hardware_setup.md)
```

#### Step 3: Update Configuration (Optional)
- Click **Edit Configuration** button
- Modal displays editable `llm_config.yaml` content
- Make changes (e.g., update URL, change model)
- Click **Save** - writes to config file
- Click **Reload** to apply changes

**Warning Displayed**:
```
⚠️ Changes require Atlas agent restart to take effect
```

#### Step 4: Cloud Endpoint Configuration (Placeholder)
- Locate **Cloud Endpoint** section (collapsed by default)
- Expand to view placeholder fields:
  - API Key: `[Enter OpenAI API Key]`
  - Model: `gpt-4`
  - Status: Disabled

**Note Displayed**:
```
ℹ️ Cloud endpoints are not yet implemented. This section is a placeholder for future integration.
```

## Tab 2: Propose

### Purpose
Generate patch proposals from error context. Supports automated workflow error parsing and manual prompt overrides.

### Walkthrough

#### Step 1: Select Input Mode
- **Automated Mode** (default): Reads from CI failure logs
- **Manual Mode**: Operator provides custom diagnostic prompt

Toggle via radio buttons:
```
○ Automated (from CI logs)
● Manual (custom prompt)
```

#### Step 2: Provide Error Context (Automated Mode)

**If Automated Mode Selected**:
- Field: **Workflow Run ID**
  - Enter GitHub workflow run ID (e.g., `12345678`)
  - Click **Fetch Logs** button
  - Atlas queries GitHub API for failure logs
  - Displays extracted error context in read-only textarea

**Example Extracted Context**:
```
Error: ModuleNotFoundError: No module named 'rocm_utils'
File: src/installer.py, Line 42
Failed Step: Install ROCm Components
Exit Code: 1
```

#### Step 3: Provide Custom Prompt (Manual Mode)

**If Manual Mode Selected**:
- Field: **Diagnostic Prompt** (large textarea)
- Operator enters custom instructions for LLM

**Example Custom Prompt**:
```
The build fails during the ROCm driver installation phase. 
Recent changes to src/installer.py may have introduced an import issue.
Focus analysis on imports in that file and suggest a fix.
```

#### Step 4: Generate Patch Proposal
- Click **Generate Patch** button
- Atlas sends request to configured LLM endpoint
- Progress indicator displayed: `🔄 Generating patch...`
- On completion, results appear in **Proposal Results** panel

**Example Results Display**:
```
✅ Patch Generated

Confidence Score: 0.87 (High)

Explanation:
The error occurs because 'rocm_utils' module import is missing. 
The fix adds the import statement at line 3 of src/installer.py.

Affected Files:
- src/installer.py

Test Commands:
- pytest tests/test_installer.py
- powershell .\build.ps1

Patch Preview:
--- a/src/installer.py
+++ b/src/installer.py
@@ -1,5 +1,6 @@
 import os
 import sys
+import rocm_utils
 
 def install_components():
```

#### Step 5: Review and Proceed
- Operator reviews confidence score, explanation, and diff
- **Actions Available**:
  - **Proceed to Verify** - Moves to Verify tab with patch loaded
  - **Edit Patch Manually** - Opens diff editor for manual modifications
  - **Regenerate** - Re-prompts LLM (e.g., if confidence too low)

**Manual Edit Flow** (if selected):
- Inline diff editor displays patch
- Operator modifies diff directly
- Click **Save Edited Patch**
- Warning displayed:
  ```
  ⚠️ Manual edit: This patch no longer reflects pure LLM output.
  Provenance log will mark as operator-modified.
  ```

## Tab 3: Verify

### Purpose
Test patches in isolated git worktrees. Run build commands and test suites. Display detailed failure information if verification fails.

### Walkthrough

#### Step 1: Load Patch
- Patch automatically loaded if arriving from Propose tab
- **OR** manually select from recent proposals dropdown:
  ```
  Recent Proposals:
  ○ atlas-patch-20251029-143022 (Confidence: 0.87, Fix import error)
  ○ atlas-patch-20251029-120515 (Confidence: 0.72, Update build script)
  ```

#### Step 2: Configure Verification
- **Worktree Name**: Auto-generated (e.g., `atlas-verify-20251029-143022`)
- **Build Command**: Read from `llm_config.yaml` for target repo
  - Display: `powershell -ExecutionPolicy Bypass -File .\build.ps1`
  - Editable if operator wants custom command
- **Test Commands**: Read from patch proposal
  - Display list:
    - `pytest tests/test_installer.py`
    - `powershell .\build.ps1`
  - Operator can add/remove commands

#### Step 3: Run Verification
- Click **Run Verify** button
- Progress displayed with live output:

```
🔄 Creating worktree: atlas-verify-20251029-143022...
✅ Worktree created

🔄 Applying patch...
✅ Patch applied successfully

🔄 Running build command...
  > powershell -ExecutionPolicy Bypass -File .\build.ps1
  [Live stdout/stderr streamed here]
✅ Build succeeded (exit code 0, duration 45.2s)

🔄 Running test commands...
  > pytest tests/test_installer.py
  [Live test output]
✅ Tests passed (exit code 0, duration 12.4s)

🎉 Verification Passed
```

#### Step 4: Review Results

**If Verification Passed**:
- Summary displayed:
  ```
  ✅ All verification steps passed
  
  Build Status: Pass
  Test Status: Pass
  Total Duration: 57.6s
  
  Next Steps:
  - Proceed to Apply tab to review and merge
  - OR trigger refinement if additional changes needed
  ```

- **Actions**:
  - **Proceed to Apply** - Moves to Apply tab
  - **Re-Verify** - Runs verification again (useful after manual edits)
  - **Clean Up Worktree** - Deletes temporary worktree

**If Verification Failed**:
- Failure details displayed:
  ```
  ❌ Verification Failed
  
  Build Status: Pass
  Test Status: Failed (1 of 2 tests failed)
  
  Failed Test:
  > pytest tests/test_installer.py
  Exit Code: 1
  Duration: 3.2s
  
  Error Output:
  ============================= FAILURES =============================
  _____________ TestInstaller.test_rocm_import ______________
  
      def test_rocm_import():
  >       from rocm_utils import validate_driver
  E       ImportError: cannot import name 'validate_driver' from 'rocm_utils'
  
  tests/test_installer.py:15: ImportError
  =========================== short test summary info ============================
  FAILED tests/test_installer.py::TestInstaller::test_rocm_import - ImportError
  ```

- **Actions**:
  - **Trigger Refinement** - Sends failure context back to LLM for new proposal
  - **Manual Edit** - Operator modifies patch directly
  - **Abort and Escalate** - Flags issue for human review, stops automation

#### Step 5: Refinement Loop (If Triggered)
- Operator clicks **Trigger Refinement**
- Failure context sent to LLM with retry prompt
- New patch proposal generated (increments retry counter)
- Displayed:
  ```
  🔄 Refinement Attempt 1 of 3
  
  Previous Patch:
  [Shows diff of failed patch]
  
  Failure Context:
  [Shows error output from verification]
  
  New Proposal:
  [Shows updated diff with explanation]
  ```

- Operator can accept new proposal and re-verify
- If max retries (default: 3) exhausted, escalation prompt displayed

## Tab 4: Apply

### Purpose
Review verified patches and merge to `Master` branch. Provides final safety checkpoint with manual confirmation.

### Walkthrough

#### Step 1: Load Verified Patch
- Patch automatically loaded if arriving from Verify tab
- **OR** manually select from verified patches dropdown:
  ```
  Verified Patches:
  ○ atlas-patch-20251029-143022 (Verified ✅, Confidence: 0.87)
  ```

#### Step 2: Review Patch Details
- Full patch summary displayed:
  ```
  Patch ID: atlas-patch-20251029-143022
  Confidence Score: 0.87
  Verification Status: ✅ Passed (57.6s)
  
  Explanation:
  Fixed import error by adding rocm_utils module import.
  
  Affected Files:
  - src/installer.py
  
  Verification Results:
  - Build: ✅ Pass (45.2s)
  - Tests: ✅ Pass (12.4s)
  ```

#### Step 3: Preview Git Commands
- Section: **Git Commands to Execute**
- Displays exact commands Atlas will run:
  ```
  1. git checkout Master
  2. git pull origin Master
  3. git apply atlas-patch-20251029-143022.diff
  4. git add src/installer.py
  5. git commit -m "atlas: fix import error in installer.py"
  6. git push origin Master
  ```

- Operator can click **Show Commit Message** to view full commit text (includes metadata)

**Example Commit Message**:
```
atlas: fix import error in installer.py

Proposed by: Atlas Agent v0.1.69
Confidence: 0.87
Triggered by: CI failure #142
Verification: All tests passed in worktree atlas-verify-20251029-143022

Affected files:
- src/installer.py

Co-authored-by: Atlas Agent <atlas@gillsystems.local>
```

#### Step 4: Manual Confirmation (Standard Flow)

**If `enable_master_push: false` in config**:
- Button: **Prepare Commit (Manual Push Required)**
- Operator clicks button
- Atlas prepares commit locally but does NOT push
- Instructions displayed:
  ```
  ✅ Commit prepared locally
  
  To push to Master, run manually:
    cd <repo_path>
    git push origin Master
  
  OR set ATLAS_PUSH_TOKEN in GitHub secrets to enable automated push.
  ```

#### Step 5: Typed Confirmation (High-Risk Mode)

**If `enable_master_push: true` in config**:
- **⚠️ Warning Banner Displayed**:
  ```
  ⚠️⚠️⚠️ MASTER PUSH ENABLED ⚠️⚠️⚠️
  
  This action will automatically push to the Master branch.
  Type the exact phrase below to confirm:
  
  [Text field: "I understand the risk and authorize Master push"]
  ```

- Operator must type exact phrase (case-sensitive)
- Button: **Execute and Push** (disabled until phrase matches)
- On match, button enabled
- Operator clicks **Execute and Push**
- Atlas executes git commands and pushes to Master
- Confirmation displayed:
  ```
  ✅ Patch applied and pushed to Master
  
  Commit Hash: abc123def456
  Push Status: Success
  Provenance Log: Updated
  
  View on GitHub: [Link to commit]
  ```

#### Step 6: Post-Apply Actions
- **Monitor CI**: Button to open GitHub Actions workflow for post-merge validation
- **View Provenance**: Link to JSONL log entry for this application
- **Rollback (If Needed)**: Link to Rollback tab with this commit pre-selected

## Tab 5: Rollback

### Purpose
Revert applied patches with full audit trail. Supports manual operator-triggered rollback and CI failure detection.

### Walkthrough

#### Step 1: View Recent Atlas Commits
- Table displays recent commits made by Atlas:

| Commit Hash | Timestamp | Confidence | Affected Files | Status | Actions |
|-------------|-----------|-----------|----------------|--------|---------|
| abc123def | 2025-10-29 14:32 | 0.87 | src/installer.py | ✅ Active | [Rollback] |
| xyz789abc | 2025-10-29 12:05 | 0.72 | build.ps1 | 🔄 Rolled Back | [View] |

#### Step 2: Select Commit to Rollback
- Operator clicks **Rollback** button for target commit
- Commit details displayed:
  ```
  Rollback Preview
  
  Commit to Revert: abc123def456
  Timestamp: 2025-10-29 14:32:10
  Patch ID: atlas-patch-20251029-143022
  
  Original Change:
  - Fixed import error in src/installer.py
  
  Verification Status at Apply: ✅ Passed
  Current CI Status: ⚠️ Failed (detected in subsequent run)
  ```

#### Step 3: Review Rollback Strategy
- Radio buttons for rollback method:
  ```
  ○ Revert Commit (preserves history, recommended)
  ○ Hard Reset (destructive, emergency only)
  ```

**Revert Commit (Default)**:
- Creates new commit that undoes changes
- Safe, preserves full history
- Recommended for normal operations

**Hard Reset (Emergency)**:
- Resets branch to state before patch
- Destructive, rewrites history
- Requires force push
- Only use if revert commit causes conflicts

#### Step 4: Preview Rollback Commands

**If Revert Selected**:
```
Git Commands to Execute:
1. git checkout Master
2. git pull origin Master
3. git revert abc123def456 --no-edit
4. git push origin Master
```

**If Hard Reset Selected**:
```
⚠️ DESTRUCTIVE OPERATION ⚠️

Git Commands to Execute:
1. git checkout Master
2. git fetch origin
3. git reset --hard xyz789abc012  (commit before patch)
4. git push origin Master --force

WARNING: This will rewrite Master branch history.
All downstream clones must force-pull.
```

#### Step 5: Provide Rollback Reason
- Field: **Rollback Reason** (required)
- Operator enters justification:
  ```
  Example:
  "Post-merge CI detected regression in test_integration.py. 
  TestIntegration::test_startup fails with ImportError."
  ```

- Reason will be included in rollback commit message and provenance log

#### Step 6: Manual Confirmation
- Checkbox: `☐ I have reviewed the rollback preview and confirm this action`
- Button: **Execute Rollback** (disabled until checkbox checked)
- Operator checks box and clicks button

**For Revert**:
- Atlas executes commands
- Progress displayed:
  ```
  🔄 Checking out Master...
  ✅ On branch Master
  
  🔄 Pulling latest changes...
  ✅ Up to date
  
  🔄 Reverting commit abc123def456...
  ✅ Revert committed
  
  🔄 Pushing to Master...
  ✅ Pushed successfully
  ```

**For Hard Reset** (if selected):
- Additional typed confirmation required:
  ```
  ⚠️ FINAL CONFIRMATION REQUIRED ⚠️
  
  Type the exact phrase to authorize destructive rollback:
  [Text field: "I authorize force push to Master"]
  ```

#### Step 7: Rollback Confirmation
- Success message displayed:
  ```
  ✅ Rollback completed
  
  Rollback ID: atlas-rollback-20251029-150315
  Revert Commit: def456abc789
  
  Provenance Log: Updated
  GitHub PR Annotation: Posted
  
  Original Patch: abc123def456
  Rollback Reason: Post-merge CI detected regression
  ```

#### Step 8: Post-Rollback Actions
- **Monitor CI**: Link to verify rollback resolved issue
- **View Rollback History**: Shows all rollbacks for this patch
- **Investigate Root Cause**: Link to original error logs for analysis

## Safety Boundaries and State Transitions

### Read-Only Mode vs. Active Mode

#### Read-Only Mode (Default on Launch)
- **Indicator**: 🔒 Read-Only Mode badge in header
- **Allowed Actions**:
  - View configurations
  - Test LLM connectivity
  - Review past patches and rollbacks
  - Generate proposals (not applied)
- **Blocked Actions**:
  - Apply patches
  - Push to Master
  - Execute rollbacks

**To Enter Active Mode**:
- Click **Enable Active Mode** button in header
- Warning modal displayed:
  ```
  ⚠️ Activate Atlas Agent?
  
  Active Mode enables patch application and rollback operations.
  All actions will be logged and require manual confirmation.
  
  [Cancel] [Activate]
  ```

#### Active Mode
- **Indicator**: ✅ Active Mode badge in header
- **All Actions Enabled**: Propose, Verify, Apply, Rollback
- **Safety Controls Still Active**: Manual confirmations still required

**To Return to Read-Only Mode**:
- Click **Disable Active Mode** button
- Confirmation modal:
  ```
  Return to Read-Only Mode?
  
  Any in-progress patches will remain available but cannot be applied.
  
  [Cancel] [Return to Read-Only]
  ```

### Typed Confirmation Flows

#### Scenario 1: Enable Master Push
**Location**: LLM Config tab → Safety Settings section

- Toggle: `enable_master_push`
- Default: OFF (requires manual git push)
- To enable:
  1. Click toggle switch
  2. Modal appears:
     ```
     ⚠️ Enable Automatic Push to Master?
     
     This allows Atlas to push commits directly to the Master branch
     without manual intervention. Use with extreme caution.
     
     Type the exact phrase to confirm:
     [Text field: "I understand the risk"]
     
     [Cancel] [Enable]
     ```
  3. Operator types phrase exactly (case-sensitive)
  4. **Enable** button activates only on exact match
  5. On enable, warning banner displayed:
     ```
     ⚠️ MASTER PUSH ENABLED ⚠️
     Atlas can now push commits automatically.
     Review all patches carefully before approval.
     ```

#### Scenario 2: Hard Reset Rollback
**Location**: Rollback tab → Hard Reset option

- Radio button: Hard Reset (destructive)
- On select:
  ```
  ⚠️⚠️⚠️ DESTRUCTIVE OPERATION ⚠️⚠️⚠️
  
  This will rewrite Master branch history and require force push.
  Only use in true emergencies.
  
  Type the exact phrase to authorize:
  [Text field: "I authorize force push to Master"]
  
  [Cancel] [Proceed]
  ```

### Warnings and Alerts

#### Low Confidence Patches
**Trigger**: Confidence score < 0.6

**Alert Displayed in Propose Tab**:
```
⚠️ Low Confidence Warning

This patch has a confidence score of 0.52 (Low).
LLM indicates uncertainty about correctness.

Recommendations:
- Review patch diff carefully before proceeding
- Consider manual refinement
- Run additional validation tests

Proceed with caution.
```

#### Verification Timeout
**Trigger**: Verification exceeds configured timeout (default: 600s)

**Alert Displayed in Verify Tab**:
```
⏱️ Verification Timeout

Verification exceeded 600 seconds and was aborted.

Possible causes:
- Build command hung or entered interactive mode
- Test suite taking longer than expected
- System resource constraints

Actions:
- Review build/test logs for hangs
- Increase timeout in llm_config.yaml if legitimately long
- Check system resources (CPU, memory, disk)
```

#### Rollback on Recently Applied Patch
**Trigger**: Attempting rollback <1 hour after apply

**Alert Displayed in Rollback Tab**:
```
⚠️ Recent Patch Rollback

This patch was applied only 23 minutes ago.

Recommendation: Allow CI to complete before rolling back.
If CI has not yet finished, rolling back may be premature.

Proceed anyway? [Cancel] [Proceed]
```

## Best Practices for Operators

### General UI Usage
1. **Always start in Read-Only Mode** - familiarize yourself with UI before enabling Active Mode
2. **Review confidence scores** - treat <0.7 as low confidence, requiring extra scrutiny
3. **Verify logs carefully** - check both build and test outputs before applying
4. **Use manual mode for complex issues** - custom diagnostic prompts often yield better patches

### Manual Override Patterns
1. **Edit patches incrementally** - small manual tweaks, not full rewrites
2. **Document manual changes** - add comments in Propose tab notes field
3. **Re-verify after editing** - never apply manually edited patch without re-running verification
4. **Escalate if uncertain** - better to pause and consult team than apply risky patch

### Safety Boundaries
1. **Never disable all safety controls** - always require at least manual confirmation
2. **Use typed confirmations seriously** - they exist to prevent accidents
3. **Monitor CI after apply** - stay available for 30+ minutes post-merge
4. **Keep rollback tab open** - quick access if CI detects issues

### State Transitions
1. **Active Mode only when needed** - return to Read-Only between operations
2. **Log out when away** - close UI if stepping away for extended period
3. **One operator at a time** - coordinate with team to avoid conflicting operations

## Troubleshooting

### Issue: UI Not Loading
**Symptoms**: Browser shows "connection refused" or blank page

**Solutions**:
1. Verify Streamlit running: Check terminal for `You can now view your Streamlit app in your browser.`
2. Confirm URL: Must be `http://127.0.0.1:8501` (not `localhost`, not other IPs)
3. Check firewall: Ensure local connections to port 8501 allowed

### Issue: LLM Config Tab Shows "Endpoint Unreachable"
**Symptoms**: Red ❌ in health check, connection test fails

**Solutions**:
1. Verify LLM server running: `curl http://127.0.0.1:8080/v1/models`
2. Check `llm_config.yaml`: Ensure URL matches actual server address
3. Review LLM server logs for errors
4. See `docs/hardware_setup.md` for LLM server troubleshooting

### Issue: Verify Tab Shows "Worktree Creation Failed"
**Symptoms**: Error creating temporary worktree

**Solutions**:
1. Check disk space: Worktrees require full repo copy
2. Verify git installed and in PATH: `git --version`
3. Ensure repo clean: `git status` should show no uncommitted changes
4. Check for stale worktrees: `git worktree list`, remove any `atlas-verify-*` manually

### Issue: Apply Tab Button Stuck on "Disabled"
**Symptoms**: Cannot click "Execute and Push" even after typing confirmation

**Solutions**:
1. Verify exact phrase: Case-sensitive, no extra spaces
2. Check Active Mode: Must be in Active Mode (not Read-Only)
3. Confirm patch verified: Must have passing verification status
4. Review logs: Check browser console for JavaScript errors
