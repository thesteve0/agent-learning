# LiteLLM Migration Guide

**Date**: March 25, 2026
**Reason**: Supply chain attack on LiteLLM package (versions 1.82.7-1.82.8)

---

## What Happened

On March 24, 2026, the LiteLLM Python package was compromised via a supply chain attack:
- **Attack vector**: Poisoned Trivy scanner in LiteLLM's CI/CD pipeline
- **Malicious versions**: 1.82.7 and 1.82.8 (published March 24, 2026)
- **Malware capabilities**:
  - Credential harvesting (SSH keys, cloud credentials, Kubernetes secrets, API keys)
  - Persistent backdoor (systemd service)
  - Lateral movement (Kubernetes pod deployment)
  - Data exfiltration (encrypted with RSA-4096 + AES-256)

**Impact**: Any installation/upgrade of LiteLLM on or after March 24, 2026 may have installed the malicious package.

---

## Course Migration

All course materials have been updated to use `OpenAIServerModel` (built into Smolagents) instead of `LiteLLMModel`.

### What Changed

**Before (compromised)**:
```python
from smolagents import LiteLLMModel

model = LiteLLMModel(
    model_id="Qwen2.5-VL-7B-Instruct",
    base_url="http://localhost:8001/v1",
    api_key="EMPTY"
)
```

**After (safe)**:
```python
from smolagents import OpenAIServerModel

model = OpenAIServerModel(
    model_id="Qwen2.5-VL-7B-Instruct",
    base_url="http://localhost:8001/v1",
    api_key="EMPTY"
)
```

### Functionality

**No functionality is lost**. `OpenAIServerModel` provides identical capabilities:
- ✅ Connects to vLLM (local or remote)
- ✅ Connects to OpenAI API
- ✅ Connects to any OpenAI-compatible endpoint
- ✅ Same parameters: `model_id`, `base_url`, `api_key`
- ✅ Works with CodeAgent and ToolCallingAgent
- ✅ Built into Smolagents (no extra dependency)

---

## Files Updated

### Python Examples
- ✅ `examples/module2_smolagents_basic.py`
- ✅ `examples/module3_smolagents_vllm.py`
- ✅ `examples/module4_production_wrapper.py`
- ✅ `examples/module7_conference_demo.py`

### Documentation
- ✅ `modules/module2-smolagents-basics.md`
- ✅ `modules/module3-vllm-integration.md`
- ✅ `modules/module4-production-wrapper.md`
- ✅ `modules/module7-conference-demos.md`

### Exercises
- ✅ `exercises/exercise2_agent_config.md`
- ✅ `exercises/exercise3_debugging.md`

### Resources
- ✅ `docs/smolagents-quickstart.md`
- ✅ `resources/smolagents-cheatsheet.md`
- ✅ `curriculum.md`
- ✅ `diagrams/codeagent-execution.txt`

---

## Safe Installation

### Recommended Installation

```bash
# Install Smolagents (without LiteLLM)
uv add 'smolagents[toolkit]'

# Verify installation
python -c "import smolagents; print(smolagents.__version__)"

# Check that LiteLLM is NOT installed
python -c "import litellm" 2>&1 | grep -q "No module" && echo "✅ LiteLLM not installed (safe)"
```

### If You Already Installed LiteLLM

**Check if you're compromised**:
```bash
# Check installed version
pip show litellm

# Look for malicious .pth file
find ~/.cache/uv -name "litellm_init.pth"
find .venv -name "litellm_init.pth"

# If found, YOU ARE COMPROMISED
```

**If compromised, take immediate action**:
1. **Quarantine the environment**:
   ```bash
   # Destroy the virtual environment
   rm -rf .venv

   # Clear uv cache
   uv cache clean
   ```

2. **Rotate ALL credentials** that were accessible from that environment:
   - SSH keys (`~/.ssh/`)
   - Cloud provider credentials (`~/.aws/`, `~/.gcp/`, `~/.azure/`)
   - Kubernetes configs (`~/.kube/`)
   - API keys in `.env` files
   - Database passwords
   - Git credentials

3. **Check for persistence**:
   ```bash
   # Look for malicious systemd service
   systemctl list-units | grep sysmon

   # If found, disable and remove
   sudo systemctl stop sysmon.service
   sudo systemctl disable sysmon.service
   sudo rm /etc/systemd/system/sysmon.service
   ```

4. **Recreate clean environment**:
   ```bash
   # Create new virtual environment
   uv venv

   # Install Smolagents (without LiteLLM)
   uv add 'smolagents[toolkit]'
   ```

---

## Completing the Course Safely

### Module 2
- Uses `InferenceClientModel` (no LiteLLM needed)
- Safe to complete as-is

### Module 3
- Now uses `OpenAIServerModel` instead of `LiteLLMModel`
- Same functionality, zero security risk
- All examples updated

### Module 4-7
- All updated to use `OpenAIServerModel`
- Production code is safe

---

## Future Updates

**When is LiteLLM safe again?**

Wait for:
1. Official all-clear from LiteLLM maintainers
2. PyPI unquarantines the package
3. New safe version released (likely 1.83.0+)
4. Independent security audit confirms package is clean

**Even then**, this course will continue using `OpenAIServerModel` because:
- ✅ One less dependency
- ✅ Built into Smolagents (no extra install)
- ✅ Same functionality
- ✅ No supply chain risk

---

## References

- [FutureSearch.ai Analysis](https://futuresearch.ai/blog/litellm-pypi-supply-chain-attack/)
- [TheHackerNews Report](https://thehackernews.com/2026/03/teampcp-backdoors-litellm-versions.html)
- [ARMO Security Analysis](https://www.armosec.io/blog/litellm-supply-chain-attack-backdoor-analysis/)
- [Snyk Analysis](https://snyk.io/articles/poisoned-security-scanner-backdooring-litellm/)

---

**Questions?** Open an issue at: https://github.com/anthropics/claude-code/issues
