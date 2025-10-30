# action-workflow-error-self-healer-agent
Don't want to click the tests each iteration for your change in code?  Yeah, annoyed me day 1 too.  My robot - Atlas fixes that irritation.  It will keep editing and committing for you until the compiling completes without errors.  Over time, we will have it self improve to increase it's speed and effectiveness.  Now or then, either way it's a win! Gill.  
=======
# Atlas Self-Healing Workflow Agent

## Overview
Atlas is an autonomous, safety-first GitHub Actions workflow error detection and self-healing agent. It monitors CI/CD pipeline failures, diagnoses root causes, and proposes or applies fixes using LLM-powered patch generation. Atlas is designed for dual-repo integration, robust audit trails, and human-in-the-loop safety controls.

---

## Key Features
- **OpenAI-compatible LLM integration** (local or cloud)
- **Propose → Verify → Refine → Apply** patch lifecycle
- **Temporary git worktree verification** (no main branch pollution)
- **Manual confirmation and rollback controls**
- **Append-only JSONL provenance logging**
- **Streamlit UI** for operator review, patch approval, and rollback
- **Windows-first workflows** (PowerShell, conda, ROCm installer integration)
- **Tested on Radeon 7900 XTX/7600 hardware**

---

## Documentation
- **Quick Start & AI Agent Guide:** `.github/copilot-instructions.md`
- **LLM API & Iteration:** [`docs/llm_integration.md`](docs/llm_integration.md)
- **Patch Lifecycle & Rollback:** [`docs/patch_lifecycle.md`](docs/patch_lifecycle.md)
- **Hardware Setup (ROCm, GPUs):** [`docs/hardware_setup.md`](docs/hardware_setup.md)
- **Streamlit UI Workflows:** [`docs/ui_workflows.md`](docs/ui_workflows.md)

---

## Getting Started
1. **Clone this repo**
2. **Set up your LLM server** (see `docs/hardware_setup.md`)
3. **Configure Atlas** (`atlas_core/config/llm_config.yaml`)
4. **Integrate with your target repo** (see `.github/copilot-instructions.md`)
5. **Start the Streamlit UI** for manual review (see `docs/ui_workflows.md`)

---

## Release Notes
### v0.1.69 (Initial Release)
- Full dual-repo architecture (agent + target integration)
- Canonical propose → verify → refine → apply patch loop
- OpenAI-compatible LLM API (local/cloud)
- Streamlit UI for patch review, manual override, and rollback
- Append-only JSONL provenance and multi-layer audit trail
- Windows-first, ROCm installer integration, tested on Radeon 7900 XTX/7600
- Modular documentation in `docs/`

---

## Contributing
- See `.github/copilot-instructions.md` for AI agent and developer onboarding
- All patches and rollbacks require manual confirmation by default
- Please open issues or PRs for feature requests and bug reports

---

## License
[MIT License](LICENSE)

---

## Maintainer
- [OCNGill](https://github.com/OCNGill)

---

For full technical details, see the `docs/` directory and `.github/copilot-instructions.md`.
>>>>>>> ec98044 (docs: ensure README.md is present and up to date)
