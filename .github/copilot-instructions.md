# Atlas Self-Healing Workflow Agent - AI Coding Instructions

## Project Overview
This is an autonomous GitHub Actions workflow error detection and self-healing agent system designed to monitor, diagnose, and automatically fix CI/CD pipeline failures. The agent operates across two repositories with safety guardrails and human oversight controls.

**Core Philosophy**: Safety-first automation that enhances rather than replaces human judgment in critical CI/CD operations.

## Quick Reference
- **Detailed Documentation**: See `docs/` for subsystem-specific guides
  - [`docs/llm_integration.md`](../docs/llm_integration.md) - LLM API contracts, schemas, iteration cycles
  - [`docs/patch_lifecycle.md`](../docs/patch_lifecycle.md) - Patch verification, rollback, audit trails
  - [`docs/hardware_setup.md`](../docs/hardware_setup.md) - GPU configurations, ROCm compatibility
  - [`docs/ui_workflows.md`](../docs/ui_workflows.md) - Streamlit UI walkthroughs, safety controls

## Architecture & Key Components

### Core Structure
- **`atlas_core/`** - Main agent logic and CLI interface
  - `tools/generate_patch.py` - Patch generation engine using LLM analysis
  - `config/llm_config.yaml` - LLM endpoints and safety settings 
  - `agents/` - Agent behavior modules (propose, verify, refine, apply)
  - `provenance/` - Append-only JSONL audit logs
  - `ui/streamlit/` - Local web UI for operator interaction (127.0.0.1 only)
- **`.github/workflows/self-healing.yml`** - CI workflow integration
- **`tests/`** - Mock LLM testing and integration validation
- **Target Integration** - Deployed as git subtree in target repos at `agent/self-heal/`

### Two-Repository Model
1. **Agent Repository** (`OCNGill/action-workflow-error-self-healer-agent`)
   - Contains core Atlas agent implementation
   - Tags: Use `v0.1.69` for initial releases
   - Branch: `Master` (capital M, not `main`)

2. **Target Repository** (`OCNGill/Radeon_Open_Compute_ROCm_Installer_Windows_x64`)
   - ROCm installer with integrated agent via git subtree
   - Build command: `powershell -ExecutionPolicy Bypass -File .\build.ps1`
   - Tags: Use `v1.3.3` for target repo versions

## Critical Development Patterns

### Canonical Workflow: Propose → Verify → Refine → Apply
1. **Propose**: LLM generates structured JSON patch with confidence score, diff, explanation, affected files, test commands
2. **Verify**: Apply patch in temporary git worktree, run build command and test suite
3. **Refine**: If tests fail, iterate with failure context (max 3 retries by default)
4. **Apply**: Manual confirmation required before merging to `Master`

See [`docs/patch_lifecycle.md`](../docs/patch_lifecycle.md) for complete verification and rollback procedures.

### Safety-First Design
```yaml
# config/llm_config.yaml - Always default to safe mode
enable_auto_apply: false
enable_master_push: false
require_manual_confirmation: true
```

**Critical Safety Controls**:
- Typed confirmation required for `enable_master_push` (operator must type exact phrase)
- All rollbacks require manual approval (CI-triggered or operator-initiated)
- Multi-layer audit trail: git commits + JSONL provenance logs + GitHub PR annotations
- Read-only mode vs. active mode state transitions are explicit in UI

### LLM Integration
- **API Format**: OpenAI-compatible REST API (`/v1/chat/completions`)
- **Response Schema**: Structured JSON with `confidence_score`, `patch_diff`, `explanation`, `affected_files`, `test_commands`
- **Local Primary**: Gillsystems 7900 XTX (24GB VRAM) hosting 13B models with quantization
- **Cloud Fallback**: Placeholder for cloud endpoints (same OpenAI-compatible schema)
- **Mock Testing**: All tests mock LLM endpoints for reproducibility

See [`docs/llm_integration.md`](../docs/llm_integration.md) for complete API contracts and iteration logic.

### Provenance & Audit Trail
- **Append-only JSONL logging** in `atlas_core/provenance/`
- Log: timestamp, agent, prompt, model/endpoint, diff, confidence, tests, verify result
- **Never** modify existing log entries
- **Rollback tracking**: Separate JSONL entries + git commit metadata + GitHub PR annotations

### Windows-First Development
- PowerShell commands in documentation and scripts
- Conda environment: `python=3.10`
- Local Streamlit UI binds to `127.0.0.1` only
- **Hardware**: See [`docs/hardware_setup.md`](../docs/hardware_setup.md) for GPU-specific configurations
  - 7900 XTX (24GB): Primary LLM host for 13B models
  - 7600 (8GB): Secondary/fallback, not for model parallelism
  - ROCm hosting documented for Linux (Windows ROCm is experimental)

## Security & Deployment Workflow

### Secret Management
```bash
# ATLAS_PUSH_TOKEN required for target repo pushes
gh secret set ATLAS_PUSH_TOKEN --body "<token>" --repo <target-repo>
```

### Git Subtree Integration
```powershell
# Add agent to target repo
git subtree add --prefix agent/self-heal https://github.com/OCNGill/action-workflow-error-self-healer-agent.git v0.1.69 --squash

# Update existing integration
git subtree pull --prefix agent/self-heal https://github.com/OCNGill/action-workflow-error-self-healer-agent.git v0.1.69 --squash
```

### Confirmation Flow
- Use `confirm_enable_master_push.py` for manual approval
- Require typed confirmation before any target repo pushes
- Operator can enable auto-apply only after explicit validation
- See [`docs/ui_workflows.md`](../docs/ui_workflows.md) for complete UI safety boundaries and manual override patterns

## Development Workflow

### Environment Setup
```powershell
conda create -n atlas python=3.10 -y
conda activate atlas
pip install -r requirements.txt
```

### Testing Strategy
- Mock all LLM endpoints in unit tests
- Test rollback mechanisms with temporary worktrees
- Validate patch iteration logic without external dependencies
- Windows runner integration tests in CI

### Streamlit UI Components
- **LLM Config Tab**: Local endpoint config, health check, cloud placeholders
- **Propose Tab**: Manual prompt override, confidence display, patch preview
- **Verify Tab**: Temp worktree testing, failing test display
- **Apply Tab**: Git command preview, manual confirmation flow
- **Rollback Tab**: Recent commit history, revert command generation

See [`docs/ui_workflows.md`](../docs/ui_workflows.md) for detailed walkthroughs of each tab.

## File Patterns & Conventions

### Configuration Structure
- `config/llm_config.yaml` - Never commit with `enable_master_push: true`
- Environment-specific overrides in local config files (gitignored)

### Patch Generation
- Output unified diff format in `suggested_patch.diff`
- Include confidence scores and human-readable explanations
- Support iterative refinement based on test failures

### Error Handling
- Graceful degradation when local LLM unavailable
- Comprehensive logging for debugging operator workflows
- Never auto-apply without explicit safety checks

## Key Commands & Operations

### Local Development
```powershell
# Run tests
pytest -q

# Start Streamlit UI
streamlit run atlas_core/ui/streamlit/main.py --server.address 127.0.0.1

# Generate patch (dry run)
python atlas_core/tools/generate_patch.py --dry-run
```

### Repository Management
```powershell
# Validate branch structure
git remote -v; git fetch --all --prune; git branch -a

# Tag releases
git tag -a v0.1.69 -m "Atlas initial tag v0.1.69"
git push origin v0.1.69  # Requires operator confirmation
```

When working on this project, always prioritize safety controls, comprehensive logging, and manual operator confirmation flows. The agent should enhance rather than replace human judgment in critical CI/CD operations.