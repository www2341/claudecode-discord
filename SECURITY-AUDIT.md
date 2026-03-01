# Security Audit Report / 보안 감사 보고서

**Date / 감사일**: 2026-03-02
**Auditor**: Claude Opus 4.6 (Automated Security Review)
**Scope / 범위**: Full codebase (18 TypeScript files, 3 tray apps, 4 shell scripts, CI, dependencies)

---

## Conclusion: No Data Exfiltration Risk / 결론: 데이터 유출 위험 없음

A comprehensive security review of the entire codebase confirms that **this project contains no data exfiltration vectors**. There are no suspicious URLs, hidden network calls, encoded data transmissions, or backdoors.

전체 소스코드를 포괄적으로 보안 검토한 결과, **이 프로젝트에는 데이터 유출(exfiltration) 벡터가 없습니다.** 수상한 URL, 숨겨진 네트워크 호출, 인코딩된 데이터 전송, 백도어가 없습니다.

---

## Data Exfiltration Verification / 데이터 유출 검증

| Verification Item | Result |
|---|---|
| External network calls | Only `fetch()` in `src/` downloads Discord attachments from `attachment.url` (Discord CDN) |
| Hardcoded URLs | All legitimate: `github.com/chadingTV/claudecode-discord`, `nodejs.org`, `deb.nodesource.com` |
| Suspicious encoding | No `btoa`, `atob`, `base64` encoding found |
| Dynamic code execution | No `eval()`, `new Function()`, `child_process.spawn/exec` in src/ |
| Environment variable transmission | `.env` vars parsed only in `config.ts` via Zod schema, never transmitted externally |
| Conversation data flow | Discord -> Claude Agent SDK (Anthropic official API) -> Discord response only. No intermediate servers |
| postinstall scripts | None in `package.json` |
| Dependency sources | All packages from `registry.npmjs.org` (verified via `package-lock.json`) |

---

## Issues Found (Unrelated to Data Exfiltration) / 발견된 이슈 (데이터 유출과 무관)

### HIGH (1)

**Path traversal in session file deletion** (`src/bot/handlers/interaction.ts:155-193`)
- `session-delete` handler uses `sessionId` directly in file path without UUID validation
- Requires `ALLOWED_USER_IDS` authentication, limited to `.jsonl` extension
- Recommendation: Add UUID regex validation for sessionId

### MEDIUM (4)

1. **rollup arbitrary file write vulnerability** - Build-time only, fix with `npm audit fix`
2. **undici decompression DoS** - discord.js dependency, low practical risk (Discord API only)
3. **Auto-approve mode** - By design, but grants unrestricted tool execution when enabled
4. **Attachment filename not sanitized** (`src/bot/handlers/message.ts:44`) - Apply `path.basename()`

### LOW (3)

- Rate limit per-user only, not per-channel
- SQLite DB file permissions not restricted
- `package-lock.json` should be committed for consistent dependencies

### INFO (3)

- `install.sh` uses `curl | bash` pattern (official NodeSource)
- Tray app updates via `git pull` without signature verification
- Windows uses temporary VBScript for background execution

---

## Security Checklist / 보안 체크리스트

- [x] No hardcoded secrets (loaded from `.env`, excluded in `.gitignore`)
- [x] No data exfiltration vectors
- [x] SQL injection prevention (all queries use parameterized prepared statements)
- [x] Authentication enforced on all endpoints (`isAllowedUser`)
- [x] All dependencies from official npm registry
- [x] No suspicious external URLs
- [x] No dynamic code execution
- [ ] Path traversal hardening recommended (session-delete, attachment filename)

---

## OWASP Top 10 Evaluation

| Category | Status | Notes |
|---|---|---|
| A01: Broken Access Control | MEDIUM | Path traversal in session-delete and attachment filename |
| A02: Cryptographic Failures | PASS | No crypto operations; discord.js handles TLS |
| A03: Injection | PASS | All SQL parameterized, no shell injection |
| A04: Insecure Design | PASS | Auto-approve is documented with user confirmation |
| A05: Security Misconfiguration | LOW | DB file permissions |
| A06: Vulnerable Components | MEDIUM | rollup and undici transitive vulnerabilities |
| A07: Auth Failures | PASS | ALLOWED_USER_IDS enforced everywhere, rate limiting present |
| A08: Software Integrity | INFO | curl-to-bash in install, git pull updates |
| A09: Logging/Monitoring | PASS | Console logging, errors caught |
| A10: SSRF | PASS | No user-controlled URL fetching |
