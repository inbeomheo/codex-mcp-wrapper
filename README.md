# Codex CLI-First Wrapper

Claude Code에서 OpenAI Codex를 활용하는 커맨드 파일. **CLI 우선, MCP 폴백** 전략.

**파일 1개**(`codex.md`)를 설치하면 Claude Code가 Codex를 버그 분석, 코드 리뷰, 리팩토링, 구현, 병렬 평가 등에 최적으로 활용합니다.

## 왜 CLI 우선인가

실측 벤치마크 결과 CLI가 MCP 대비 거의 모든 시나리오에서 우월합니다:

| 시나리오 | CLI 병렬 | MCP 직렬 | 배율 |
|---------|---------|---------|------|
| 평가 3개 동시 | 14초 | ~30초 | **2x** |
| 경량 호출 5개 | 6초 | ~35초 | **6x** |
| 경량 호출 10개 | 10초 | ~70초 | **7x** |

**핵심 발견:**
- MCP는 직렬 큐 — 여러 호출이 동시에 들어와도 순차 처리
- CLI는 `& + wait`로 진짜 OS-level 병렬 실행
- CLI도 `exec resume`으로 멀티턴 대화 가능 (MCP 전용 기능이 아님)
- CLI 전용 기능: `codex review`, `codex apply`, `-o` 파일 출력

## 기능 비교

| 기능 | CLI | MCP |
|------|-----|-----|
| **외부 병렬 실행** | ✅ `& + wait` | ❌ 직렬 큐 |
| **멀티턴 대화** | ✅ `exec resume` | ✅ `codex-reply` |
| **파일 읽기/쓰기** | ✅ | ❌ |
| **코드 리뷰** | ✅ `codex review` | ❌ |
| **diff 적용** | ✅ `codex apply` | ❌ |
| **출력 파일 저장** | ✅ `-o` | ❌ |
| **base-instructions** | ⚠️ config.toml | ✅ 파라미터 |
| **Agent 내부 호출** | ⚠️ Bash 경유 | ✅ 도구 직접 |

## 3원칙

1. **Claude는 지휘, Codex는 실행.** Claude가 판단, Codex가 코드를 읽고/쓰고/실행.
2. **병렬은 CLI `& + wait`로.** MCP는 직렬 큐. CLI는 진짜 병렬.
3. **용도에 맞는 서브커맨드.** `exec`(범용), `review`(리뷰), `exec resume`(멀티턴), `apply`(diff 적용).

## 설치

### 1. Codex CLI 설치 + 로그인

```bash
npm install -g @openai/codex
codex login
```

### 2. 커맨드 파일 설치

```bash
# 전역 (모든 프로젝트에서 사용)
mkdir -p ~/.claude/commands
cp codex.md ~/.claude/commands/codex.md

# 프로젝트 전용
mkdir -p .claude/commands
cp codex.md .claude/commands/codex.md
```

### 3. (선택) MCP 서버 등록

CLI만으로 충분하지만, `base-instructions` 파라미터가 필요한 경우:

```bash
claude mcp add codex -- codex mcp-server
```

## 사용법

```
/codex review              # 현재 브랜치 코드 리뷰
/codex challenge            # adversarial 모드로 코드 깨기 시도
/codex 이 함수 분석해줘     # 자유 질문 (consult)
/codex 병렬로 10개 평가해줘  # 병렬 평가 (autoresearch 등)
```

## 지뢰밭

| 함정 | 대응 |
|------|------|
| 파일 범위 미지정 → 엉뚱한 파일 수정 | 수정 대상 파일 목록 명시 |
| 같은 파일 병렬 수정 → 충돌 | 파일 소유권 배타적 배분 |
| Codex 결과 맹신 | Critical은 Read로 교차 검증 |
| 셸 특수문자 깨짐 | heredoc 또는 임시 파일 |
| MCP 직렬 큐로 N배 느림 | CLI `& + wait` 병렬 사용 |

## 모드 선택 가이드

| 상황 | 추천 | 이유 |
|------|------|------|
| 대부분의 작업 | **CLI** | 빠르고 병렬 가능 |
| 병렬 분석/평가 | **CLI** | MCP는 직렬, CLI는 진짜 병렬 |
| 코드 리뷰 | **CLI** | `codex review` 전용 서브커맨드 |
| `base-instructions` 필요 | **MCP** | CLI는 config.toml로 대체 |
| Agent 도구 내부에서 호출 | **MCP** | Bash 없이 직접 호출 편의 |

## 라이선스

MIT
