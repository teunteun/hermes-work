# 세션 작업 리포트 — 로컬 LLM 정리 · 헤르메스 코딩 브리지 구축

- **일자:** 2026-07-31
- **작업 노드:** `.56`(DGX Spark) · `.82`(hphermes) · `.78`(i9gtk) · `.68`(NUC, 오케스트레이터)
- **요약:** DGX 모델 상주 정책 정리 → hermes 백엔드 모델 교정 → `.78`에 텔레그램/CLI/원격창 공용 코딩 위임 브리지(`/code` → `hermes-do` → claude CLI → hermes-work push) 구축·검증

---

## 1. `.56` DGX Spark — Ollama 상주 정책 정리

- **하드웨어 실체 규명:** GPU = NVIDIA GB10(DGX Spark), CPU/GPU 공유 **통합메모리 121.7 GiB**. `nvidia-smi`는 memory.total을 N/A로 보고 → ollama 저널로 확인해야 함.
- **어제 속도저하의 진짜 원인:** 모델 경합이 아니라 **용량 고갈**. qwen3.5(75.8GB)+gemma4(25GB)+zeomed2(25GB)=126GB > 121.7GiB. 저널에 `available=2.0 GiB` 기록(07-29 02:28).
- **모델 구조 판별(`/api/show`):** `gemma4:26b`=MoE(expert 128중 8활성)·vision 있음, `zeomed2:31b`=dense·**vision 없음**. zeomed2는 gemma4의 상위집합이 아님. 속도차의 정체가 MoE.
- **실측:** zeomed2 단독 생성 10.6 tok/s(대역폭 273GB/s 종속). 과거 문서의 "gemma4 59.3 tok/s"(MoE)와 직접 비교 금지.
- **조치:** `/etc/systemd/system/ollama.service.d/environment.conf`
  - `OLLAMA_MAX_LOADED_MODELS` 3→**1**, `OLLAMA_KEEP_ALIVE` -1→**30m** (백업 `environment.conf.bak-20260731`)
  - 결과: `ollama ps` UNTIL이 `Forever`→시각 표시. 동시상주로 인한 용량 고갈 구조적으로 차단. (영상 수요 없음 확인 → 1로 결정)
- **추론엔진 재평가:** vLLM 재도입 검토 → **Ollama 유지 결론**. 과거 vLLM 3.6tok/s는 BF16 때문이고, 개선분은 엔진이 아니라 Q4 양자화에서 나옴. GB10은 sm_121로 FlashAttention 미지원. 전환비용 큼(EMR 4파일·hermes·정밀진단이 11434에 연결). 배치 동시요청 수요 생기면 재검토.

## 2. `.82` hphermes — 백엔드 모델 교정

- qwen3.5 삭제로 `.82`의 `~/.hermes/config.yaml` default가 `model not found`로 깨져 있었음.
- default `qwen3.5:122b-a10b` → **`zeomed2:31b-q4km`** 교체(백업 `config.yaml.bak-20260731-model`), gateway 재시작.
- 검증: 정상 응답 + **tool calling 정상**. thinking 오버헤드로 한 턴 30~60초(정상). `/v1`에선 thinking 못 끔.
- **주의:** `.82`는 **김선희 국장 소유** — 헤르메스 통합/이전 대상에서 제외.

## 3. `.78` i9gtk — 코딩 위임 브리지 구축 (핵심)

`.68`엔 claude CLI가 없어 헤르메스가 개발을 위임하지 못하던 문제를, 이미 CLI가 깔린 `.78`로 정리.

### 3-1. 인증
- OAuth 토큰은 `~/.config/claude-oauth.env`(`CLAUDE_CODE_OAUTH_TOKEN`). gateway 유닛에 `EnvironmentFile`+드롭인 `oauth.conf`로 이미 주입돼 있었음("Not logged in"은 토큰 미로드 셸에서만 보이는 착시).
- `~/.bashrc`에 토큰 source 추가 → **`ssh i9gtk` 대화형에서도 `claude` 인증됨**.

### 3-2. `hermes-do` — 단일 브리지 (`~/.local/bin/hermes-do`, 별칭 `hd`)
- `claude -p` 위임(superpowers+gstack 자동) → `hermes-work/jobs/<id>/` 산출물 + `reports/<id>.md` → `teunteun/hermes-work` push.
- **ASCII 슬러그**(한글은 sha1 해시 폴백)로 파일명 안 깨짐(옛 `…파일�.md` 문제 해결).
- allowedTools = `Read,Edit,Write,Bash,Glob,Grep,Skill,SlashCommand,Task,WebFetch,WebSearch` — **`Skill` 필수**(없으면 gstack이 스텁만 뜸).
- 토큰 자동 주입 → CLI/원격창에서도 인증 걱정 없음.

### 3-3. `/code` 슬래시명령
- 스킬 `~/.hermes/skills/software-development/code/SKILL.md`(name: code). `/code <작업>` → 백그라운드로 `hermes-do --src hermes` 위임.
- `!claude` 폐기(사용자 요청). quick_commands `type: exec`는 30초 타임아웃+인자 미전달이라 부적합 → 스킬 방식 채택.

### 3-4. 공유 모델 (요구사항: 어디서 명령하든 산출물 공유)
- 텔레그램 `/code` · ssh cli `hd` · 원격창 `hermes-do --src window` **모두 → `hermes-work` repo**로 수렴. 누구든 pull하면 3곳 산출물 공유.

### 3-5. 문서/도구
- render-doc·pandoc·weasyprint·google-chrome 모두 설치 확인(문제는 PATH뿐). PDF = `render-doc <md>`.
- `gh` CLI v2.45.0 설치(apt).

## 4. 검증 (실제 실행으로 증명)

| 항목 | 결과 | 표식/근거 |
|---|---|---|
| hermes-do 엔드투엔드(cli) | ✅ | 파일 생성 + ASCII 리포트 + push |
| 텔레그램 `/code` 실사용 | ✅ | 원장님이 보낸 실제 메시지 → `bridge-check-9931.md`, 커밋 `90ab60b` (`TTBRIDGE-20260731-9931`) |
| superpowers 스킬 호출 | ✅ | `superpowers=OK` |
| gstack 스킬 호출 | ✅ | `gstack=OK` (Skill 도구 허용 후) |
| claude가 직접 GitHub push | ✅ | origin/master에 `cap-…md` 존재, 커밋 `b18909c` (`CAPCHECK-20260731-7742`) |

## 5. 보류/후속

- **PR/gh 인증 — 보류**(사용자 결정). `.68`의 gh 토큰은 **만료**(Bad credentials)라 재사용 불가. git push는 SSH 키로 정상이므로 "코드 올리기"는 이미 완비. PR/`ship` 스킬 쓰려면 새 토큰 또는 `gh auth login` 필요.
- **`.68` 헤르메스 → `.78` 이전**은 향후 과제(자산: `~/.hermes` 설정·스킬·메모리·cron). 봇 토큰은 별개(`.68`=hermes choi, `.78`=i9gtk)라 충돌 없음.
- **보안 참고:** 오전 헤르메스 세션 로그에 `.56` 비밀번호 평문 잔존. `.68↔.56`은 이미 SSH 키 인증됨(sshpass 불필요했음).

---
*생성: 최용재 원장 / .68 오케스트레이터 세션. 정정: 이 리포트는 초판 ollama 보고서(emr-auto f2fe371)의 확장·통합본이며, .78 브리지 내용을 추가함.*
