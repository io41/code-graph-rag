# Security Audit Report — 2026-04-01

## Scope

Repository: `code-graph-rag`

Reviewed surfaces:
- application code
- MCP server and tools
- grammar installation flow
- dependency manifests / lockfile
- GitHub Actions workflows

## Methods Used

- static code review of MCP, file IO, shell, parser, provider, and grammar-management paths
- pattern search for suspicious execution, exfiltration, secrets, and unsafe dependency sources
- focused `bandit` runs
- workflow inspection for third-party action and permission risk
- targeted regression tests for security fixes

## Conclusion

No obvious intentionally malicious code, hidden backdoor, credential stealer, or obfuscated payload was found in the repository.

However, the repository originally had several genuine security weaknesses. The highest-confidence issues identified during review have now been hardened in the working tree.

## Findings

### Previously High Risk — Now Mitigated

1. **Paginated MCP file read path traversal**
   - Affected area: `codebase_rag/mcp/tools.py`
   - Risk: caller could escape the project root when using paginated reads.
   - Status: fixed by validating resolved paths against project root before opening files.

2. **Unsafe default HTTP MCP bind**
   - Affected area: `codebase_rag/config.py`, `codebase_rag/cli.py`
   - Risk: HTTP MCP server defaulted to publicly reachable bind behavior.
   - Status: fixed by defaulting to loopback (`127.0.0.1`).

3. **Remote HTTP MCP without auth**
   - Affected area: `codebase_rag/mcp/server.py`
   - Risk: remote HTTP MCP exposure without authentication.
   - Status: fixed by requiring:
     - explicit remote-bind opt-in
     - `MCP_HTTP_AUTH_TOKEN` for non-loopback binds
     - bearer-token request auth when token is configured

4. **Custom grammar repository trust**
   - Affected area: `codebase_rag/tools/language.py`
   - Risk: untrusted grammar repositories can lead to build-time code execution.
   - Status: hardened by requiring:
     - exact GitHub HTTPS repo-root URLs
     - explicit `--trust-custom-grammar-url` for custom repos
     - or exact allowlist membership via `TRUSTED_GRAMMAR_REPOS`

### Medium / Ongoing Risk

5. **Trusted grammar repos still execute build code**
   - Installing a grammar may still execute code from a trusted repository during build/setup.
   - This is now user-gated and allowlistable, but not eliminated.

6. **Third-party GitHub Actions / external review services**
   - Examples:
     - `anthropics/claude-code-action`
     - several third-party marketplace actions
   - Most actions are pinned, which is good, but they remain supply-chain trust points.

7. **Shared-token HTTP auth**
   - Current HTTP MCP protection is bearer-token based.
   - This is materially better than unauthenticated exposure, but still not per-user auth.

8. **Workflow hardening improvements applied**
   - `build-binaries.yml` no longer mutates dependencies at runtime with `uv add`; it now uses a frozen sync from lockfile.
### Low / Expected Findings

9. **Subprocess usage in grammar tooling**
   - `git submodule`, `git rm`, and related commands are still present.
   - These remain low-severity bandit findings and are expected given the feature.

10. **Hugging Face model download trust**
   - Semantic embedding model download still depends on upstream model trust and pinning posture.
   - This was identified earlier as a supply-chain consideration, not evidence of malware.

## Dependency / Lockfile Notes

- No git-based dependency sources were found in `uv.lock`.
- No external path-based dependency sources were found in `uv.lock`.
- Lockfile entries resolve from standard PyPI file URLs.

## Workflow Notes

Positive observations:
- most GitHub Actions are commit-pinned
- security scanning exists (`osv-scanner`, `scorecard`)
- release signing exists (`cosign`)

Residual concerns:
- some third-party actions remain trusted execution surfaces
- `pull_request_target` workflow usage deserves ongoing scrutiny even though the current logic appears limited

## Verification Evidence

Focused regression suites passed after hardening:

```bash
uv run pytest \
  codebase_rag/tests/test_provider_configuration.py \
  codebase_rag/tests/test_language_tool_unit.py \
  codebase_rag/tests/test_mcp_server.py \
  codebase_rag/tests/test_mcp_read_file.py -q
```

Result:
- `78 passed, 1 warning`

Focused bandit checks reported only low-severity expected findings in grammar subprocess code after hardening.

## Recommended Next Steps

1. Consider adding commit pin / revision pin support for trusted grammar repositories.
2. Consider stronger HTTP MCP auth options if remote exposure is a real use case.
3. Periodically review third-party GitHub Actions and prune any that are not essential.
4. Consider documenting an operational recommendation to avoid remote HTTP MCP unless absolutely necessary.

## Overall Security Posture

**Current assessment: Moderate, improved from weaker-by-default.**

- No malware found.
- Main externally exploitable local design issues reviewed here were addressed.
- Remaining risk is primarily supply-chain trust and optional remote-exposure configuration, not covert malicious behavior.
