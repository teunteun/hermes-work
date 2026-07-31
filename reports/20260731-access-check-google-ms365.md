# 접속 확인 리포트 — Google Workspace + MS365 outlook.kr

- **일자:** 2026-07-31
- **노드:** `.68` (NUC, amarkant) — 접속 도구 보유 노드
- **요청:** Gmail / Google Drive / Google Sheets / Google Calendar / amarkant@outlook.kr 접속 가능 여부 확인
- **결과:** **5개 서비스 전부 접속 가능** (outlook.kr은 이날 재인증 수행)

---

## 확인 결과

| 서비스 | 상태 | 실측 근거 |
|---|---|---|
| Gmail (allergyfaculty@gmail.com) | ✅ | 최신 메일 읽힘(금융결제원, 07-31 15:41 수신) |
| Google Drive | ✅ | 파일 목록 반환(`의사스케줄2026` 등) |
| Google Sheets | ✅ | 시트 셀 읽힘 |
| Google Calendar | ✅ | 당직표 이벤트 반환(`[야간] 최용재`,`[달빛] 최용재`), 캘린더 6종 접근 |
| outlook.kr (MS365) | ✅ | 재인증 후 `userPrincipalName: amarkant@outlook.kr` 확인 |

## 도구/경로 (재현용)

### Google Workspace
- **인터프리터 함정:** 시스템 python3엔 googleapiclient 없음 → 반드시 `~/.hermes/gws-venv/bin/python` 사용.
- CLI: `$PY ~/.hermes/skills/productivity/google-workspace/scripts/google_api.py <service> <verb>`
  - `gmail search "is:unread" --max N`
  - `calendar list --all --max N` (primary는 비어 있어 **`--all`/`--calendar` 필수**; 별칭 primary/hospital/tonton/duty/radiology/family)
  - `drive search "<q>" --max N`
  - `sheets get <sheet_id> <range>`
- 토큰: `~/.hermes/google_token.json` (refresh_token 보유 → 자동 갱신). 스코프: gmail·drive·calendar·spreadsheets·documents·contacts.

### MS365 outlook.kr
- 정식 경로: hermes MCP `@softeria/ms-365-mcp-server` (tenant=consumers). IMAP 토큰(`~/.hermes/.outlook-oauth.json`)은 없음.
- **문제:** 개인계정 리프레시 토큰 만료됨 → `--verify-login`이 `No valid token found` 반환(마지막 성공 7/28).
- **조치(2026-07-31, device-code 재인증):**
  ```bash
  export PATH="$HOME/.hermes/node/bin:$PATH"     # node가 이 PATH에만 있음
  export MS365_MCP_TENANT_ID=consumers
  npx -y @softeria/ms-365-mcp-server --login      # 8자리 코드 → microsoft.com/link 에서 승인
  ```
  → 로그인 성공(`"Login successful"`, Yong Jae Choi / amarkant@outlook.kr), 토큰 저장.
- 재발 시(몇 달 뒤 만료) 동일 절차로 재인증.

## 후속
- Google은 자동 갱신으로 상시 유지.
- outlook.kr은 개인계정 특성상 주기적 재인증 필요 — 위 device-code 절차 사용.
