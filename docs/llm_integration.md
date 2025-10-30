# LLM Integration Guide

## Overview
Atlas communicates with LLM endpoints using an **OpenAI-compatible API format** over REST. This ensures interoperability with local models (LM Studio, Ollama, vLLM) and cloud providers without rewriting orchestration logic.

## API Contract

### Endpoint Format
- **Protocol**: REST over HTTP
- **Schema**: OpenAI `/v1/chat/completions`
- **Serialization**: UTF-8 encoded JSON
- **Error Format**: OpenAI-style error objects

```json
{
  "error": {
    "message": "string",
    "type": "string",
    "param": "string",
    "code": "string"
  }
}
```

### Request Structure
```json
{
  "model": "string",
  "messages": [
    {"role": "system", "content": "string"},
    {"role": "user", "content": "string"}
  ],
  "temperature": 0.7,
  "max_tokens": 2048
}
```

### Response Structure for Patch Generation

Atlas requires **structured JSON** responses for all patch operations. Free-form output is rejected to ensure determinism and reliability.

**Canonical Schema v1**:
```json
{
  "confidence_score": 0.85,
  "patch_diff": "--- a/file.py\n+++ b/file.py\n@@ -1,3 +1,3 @@\n...",
  "explanation": "Human-readable rationale for the patch",
  "affected_files": ["src/module.py", "tests/test_module.py"],
  "test_commands": ["pytest tests/test_module.py", "powershell .\\build.ps1"]
}
```

**Field Definitions**:
- `confidence_score` (float, 0.0-1.0): LLM's self-assessed confidence in patch correctness
- `patch_diff` (string): Unified diff format, directly applicable via `git apply`
- `explanation` (string): Plain-text rationale (only prose element in schema)
- `affected_files` (array[string]): File paths touched by the patch
- `test_commands` (array[string]): Shell commands to validate the patch

### Schema Validation
- Atlas validates all responses against the schema before processing
- Invalid responses trigger a retry with a corrective system prompt
- Max validation failures: 3 (configurable in `llm_config.yaml`)

## Iteration Cycle: Propose → Verify → Refine → Apply

### 1. Propose Phase
**Goal**: Generate initial patch from error context

**Input**: 
- Workflow failure logs
- Repository context (recent commits, file structure)
- Error-specific diagnostic info

**Output**: Structured JSON patch (see schema above)

**LLM Prompt Pattern**:
```
You are diagnosing a CI/CD failure. Analyze the error logs and propose a patch.

Error Context:
{error_logs}

Repository State:
{repo_context}

Return your response as JSON with fields: confidence_score, patch_diff, explanation, affected_files, test_commands.
```

### 2. Verify Phase
**Goal**: Test patch in isolated environment

**Process**:
1. Create temporary git worktree
2. Apply `patch_diff` via `git apply`
3. Run target repo's build command (e.g., `.\build.ps1`)
4. Execute all `test_commands` from LLM response
5. Collect stdout/stderr and exit codes

**Success Criteria**: All commands exit with code 0

### 3. Refine Phase
**Goal**: Iterate on failed patches with failure context

**Triggered When**: Verify phase fails (build errors or test failures)

**Retry Logic**:
- Default max retries: 3 (configurable)
- Each retry includes previous failure context
- Low confidence scores (<0.5) may trigger immediate escalation

**Refinement Prompt Pattern**:
```
Your previous patch failed verification. Review the failure and propose a corrected patch.

Previous Patch:
{previous_patch_diff}

Failure Output:
{test_failure_logs}

Build Errors:
{build_errors}

Return a new JSON response with the corrected patch.
```

**Convergence Criteria**:
- Tests pass → proceed to Apply
- Max retries exhausted → escalate to manual review
- Confidence score drops below threshold → abort and escalate

### 4. Apply Phase
**Goal**: Manual confirmation before merging to `Master`

**Process**:
1. Present verified patch to operator via Streamlit UI
2. Show test results, confidence score, explanation
3. Require explicit approval (typed confirmation for high-risk actions)
4. Merge patch with provenance metadata
5. Log to JSONL and git commit message

**Never auto-apply**: All patches require human confirmation, even after successful verification.

## Configuration

### `config/llm_config.yaml` Structure
```yaml
llm_endpoints:
  local:
    url: "http://127.0.0.1:8080/v1/chat/completions"
    model: "codellama-13b"
    enabled: true
  cloud:
    url: "https://api.openai.com/v1/chat/completions"
    model: "gpt-4"
    enabled: false
    api_key_env: "OPENAI_API_KEY"

iteration:
  max_retries: 3
  confidence_threshold: 0.5
  timeout_seconds: 300

safety:
  enable_auto_apply: false
  enable_master_push: false
  require_manual_confirmation: true
```

## Error Handling

### LLM Endpoint Unavailable
- Log error to `atlas_core/logs/`
- Attempt cloud fallback if configured
- Surface error in Streamlit UI
- Do not block operator manual workflows

### Invalid Response Schema
- Log validation errors with full response
- Increment retry counter
- Send corrective prompt with schema example
- After max retries, present raw response to operator for manual intervention

### Timeout Handling
- Default: 300 seconds per LLM call
- Configurable per endpoint in `llm_config.yaml`
- On timeout: abort current operation, log, notify operator

## Extensibility

### Custom Metadata Fields
If Atlas needs richer context (e.g., tool invocation traces), extend via:
- `system` role messages with structured metadata
- Custom `metadata` fields in request (preserved in response)
- Still wrapped in OpenAI-compatible schema for transport

### Future Enhancements
- **Streaming responses**: For real-time patch generation progress
- **Multi-model orchestration**: Route different tasks to specialized models
- **Fine-tuning integration**: Custom models trained on historical Atlas patches

## Testing

### Mock LLM Endpoints
All unit tests must mock LLM endpoints to avoid external dependencies.

**Example** (`tests/test_iteration_logic.py`):
```python
def test_propose_verify_refine_cycle(mock_llm_endpoint):
    mock_llm_endpoint.return_value = {
        "confidence_score": 0.9,
        "patch_diff": "...",
        "explanation": "Fixed import error",
        "affected_files": ["src/module.py"],
        "test_commands": ["pytest tests/"]
    }
    
    result = atlas.propose_patch(error_context)
    assert result["confidence_score"] == 0.9
```

### Integration Tests
- Test against real local LLM endpoint (manual, not in CI)
- Validate schema compliance across multiple model providers
- Verify timeout and error handling with deliberate failures
