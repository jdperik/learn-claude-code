# Security Scan Report

**Project:** learn-claude-code
**Date:** 2026-03-16
**Scanner:** Claude Code Security Audit
**Overall Risk Level:** LOW (educational project)

---

## Summary

This is an educational project teaching AI agent architecture. It is **not intended for production deployment**. The codebase is generally clean with no critical vulnerabilities, but there are several security considerations worth noting, particularly around command injection in the Python agents.

| Category | Findings | Severity |
|----------|----------|----------|
| Command Injection | Shell=True subprocess usage | MEDIUM |
| XSS (Cross-Site Scripting) | dangerouslySetInnerHTML without sanitization | LOW |
| Secrets / Credentials | No hardcoded secrets found | PASS |
| Dependency Vulnerabilities | npm audit: 0 vulnerabilities | PASS |
| Path Traversal (minimal-agent.py) | Missing `safe_path()` in template | MEDIUM |
| Path Traversal (MessageBus) | Unsanitized `to` param in inbox path | LOW |
| Configuration Security | Proper .gitignore, no sensitive files | PASS |

---

## Detailed Findings

### 1. Command Injection via `subprocess.run(shell=True)` — MEDIUM

**Affected files (all agent files):**
- `agents/s01_agent_loop.py:58`
- `agents/s02_tool_use.py:52`
- `agents/s03_todo_write.py:103`
- `agents/s04_subagent.py:57`
- `agents/s05_skill_loading.py:128`
- `agents/s06_context_compact.py:135`
- `agents/s07_task_system.py:141`
- `agents/s08_background_tasks.py:68,125`
- `agents/s09_agent_teams.py:266`
- `agents/s10_team_protocols.py:307`
- `agents/s11_autonomous_agents.py:385`
- `agents/s12_worktree_task_isolation.py:55,238,252,357,380,489`
- `agents/s_full.py:84,341`

**Description:** All agent files use `subprocess.run(command, shell=True, ...)` to execute bash commands. The commands come from the LLM (Claude API) tool calls, which means the LLM decides what shell commands to run.

**Mitigation already present:** A basic blocklist exists in `s01_agent_loop.py:54`:
```python
dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
```

**Risk assessment:** MEDIUM — This is by design for an AI agent that executes shell commands. However, the blocklist is trivially bypassable (e.g., `rm -rf /*` bypasses `rm -rf /`). For an educational project, this is acceptable. For production use, a proper sandboxing approach (containers, seccomp, etc.) would be needed.

**Recommendation:** Add a note in documentation that these agents should only be run in sandboxed environments (containers, VMs) and never with elevated privileges.

---

### 2. `dangerouslySetInnerHTML` without Sanitization — LOW

**Affected files:**
- `web/src/components/docs/doc-renderer.tsx:87` — Renders markdown as raw HTML
- `web/src/app/[locale]/layout.tsx:41` — Inline script for dark mode detection

**Description:**

**doc-renderer.tsx:** The markdown rendering pipeline uses `rehype-raw` with `allowDangerousHtml: true` and outputs via `dangerouslySetInnerHTML` without using `rehype-sanitize`. If a malicious markdown document were added to the `docs/` folder, it could inject arbitrary HTML/JS.

**layout.tsx:** The inline script for dark mode detection is a static string with no user input — this is safe.

**Risk assessment:** LOW — The markdown content comes from static files in `docs/` and `data/generated/docs.json` that are committed to the repository, not from user input. An attacker would need commit access to exploit this.

**Recommendation:** Consider adding `rehype-sanitize` to the markdown pipeline as defense-in-depth:
```
npm install rehype-sanitize
```
```js
import rehypeSanitize from "rehype-sanitize";
// Add to pipeline: .use(rehypeSanitize)
```

---

### 3. Path Traversal in `minimal-agent.py` Template — MEDIUM

**Affected file:**
- `skills/agent-builder/references/minimal-agent.py:79-92`

**Description:** The `minimal-agent.py` reference template uses `WORKDIR / args["path"]` directly for `read_file` and `write_file` tools without any path validation. Unlike all main agent files (s02-s12, s_full) which include a `safe_path()` function that resolves and validates paths stay within the working directory, this template lacks that protection.

**Example exploit:** An LLM tool call with `{"path": "../../etc/passwd"}` would read files outside the working directory.

**Risk assessment:** MEDIUM — Users may copy this template as a starting point for their own agents, inheriting the vulnerability.

**Recommendation:** Add `safe_path()` validation matching the pattern used in other agent files:
```python
def safe_path(p: str) -> Path:
    resolved = (WORKDIR / p).resolve()
    if not str(resolved).startswith(str(WORKDIR.resolve())):
        raise ValueError(f"Path escapes working directory: {p}")
    return resolved
```

---

### 4. Path Traversal in `MessageBus.send()` — LOW

**Affected files:**
- `agents/s09_agent_teams.py:94` — `self.dir / f"{to}.jsonl"`
- `agents/s10_team_protocols.py:104`
- `agents/s11_autonomous_agents.py:97`
- `agents/s_full.py:373`

**Description:** The `MessageBus.send()` method constructs file paths using the `to` parameter (teammate name) without sanitization. If the LLM generates a tool call with `to` set to `"../../etc/evil"`, it would write a `.jsonl` file outside the intended inbox directory.

**Risk assessment:** LOW — The `to` parameter comes from the LLM, and teammate names are typically constrained by the team configuration. However, there is no enforcement at the `MessageBus` level.

**Recommendation:** Validate that teammate names contain only alphanumeric characters and underscores:
```python
import re
if not re.match(r'^[a-zA-Z0-9_]+$', to):
    return f"Error: Invalid recipient name '{to}'"
```

---

### 5. Secrets and Credentials — PASS

- `.env.example` contains only placeholder values (`sk-ant-xxx`)
- No actual `.env` file is committed to the repository
- `.gitignore` properly excludes `.env`, `.envrc`, and other sensitive files
- No hardcoded API keys, passwords, or tokens found in source code
- Environment variables are loaded via `python-dotenv` at runtime

---

### 6. Dependency Vulnerabilities — PASS

- **npm audit:** 0 vulnerabilities found
- **Python dependencies:** Only 2 packages (`anthropic>=0.25.0`, `python-dotenv>=1.0.0`), both well-maintained
- All dependencies use minimum version constraints, allowing security patches

---

### 7. Configuration Security — PASS

- `.gitignore` is comprehensive (excludes `.env`, `node_modules`, build artifacts, task outputs)
- GitHub Actions CI (`ci.yml`, `test.yml`) uses pinned Node 20 and Python 3.11
- Vercel configuration contains only public redirect rules
- No overly permissive CORS or CSP headers found

---

### 8. Additional Observations

| Item | Status |
|------|--------|
| No `eval()` or `exec()` usage | PASS |
| No SQL queries (no database) | N/A |
| No authentication system | N/A (static educational site) |
| No user file upload functionality | PASS |
| No SSRF vectors (no user-controlled URL fetching) | PASS |
| MIT License — no legal concerns | PASS |

---

## Conclusion

This project is **safe for its intended purpose** as an educational resource. The main security considerations are architectural (shell command execution in AI agents) rather than bugs. The web frontend is a static documentation site with no user input processing.

**Key recommendations:**
1. Run AI agents only in sandboxed environments (Docker, VMs)
2. Add `safe_path()` validation to the `minimal-agent.py` template to match other agent files
3. Sanitize teammate names in `MessageBus.send()` to prevent path traversal
4. Consider adding `rehype-sanitize` to the markdown pipeline as defense-in-depth
5. Improve the command blocklist or replace it with a proper allowlist/sandbox approach in production scenarios
