---
name: codex
description: Codex 하이브리드 래퍼 (MCP + CLI). 버그분석, 코드리뷰, 리팩토링, 구현, 질문 등 다목적 Codex 활용. MCP 연결 시 멀티턴·구조화 파라미터 활용, MCP 미연결 시 CLI 자동 폴백.
---

# Codex Hybrid Wrapper

## 모드 감지 (매 호출 시 자동 판별)

```
IF mcp__codex__codex 도구가 사용 가능:
  → MCP 모드 (멀티턴, 구조화 파라미터, base-instructions 지원)
ELSE IF `command -v codex` 성공:
  → CLI 모드 (단발성, Bash 실행, stdout 파싱)
ELSE:
  → "Codex가 설치되어 있지 않습니다" 안내 후 중단
```

## 기본 설정

```
model: "gpt-5.4"
config: {"reasoningEffort": "high"}
sandbox: "danger-full-access"
```

경량 작업(단순 질문, 짧은 확인): `model: "gpt-5.4"` + `reasoningEffort: "medium"`

## 3원칙

1. **Claude는 지휘, Codex는 실행.** Claude가 무엇을 할지 판단하고, Codex가 실제 코드를 읽고/쓰고/실행한다. Claude 컨텍스트에 코드를 넣지 말고 Codex에게 직접 다루게 하라.
2. **병렬은 Codex 내부에서.** MCP 모드: `mcp__codex__codex`를 여러 개 동시 호출해도 MCP가 직렬 처리한다. CLI 모드: 여러 `codex exec` 호출도 순차 실행된다. 병렬이 필요하면 Codex에게 셸 병렬(`& + wait`)이나 내장 멀티에이전트를 쓰라고 지시하라.
3. **Codex 능력을 최대한 활용하라.** Codex는 단순 코드 생성기가 아니다. 아래 능력 목록을 보고 위임할 수 있는 건 위임하라.

## Codex 능력

| 능력 | 설명 | 활용 예 |
|------|------|---------|
| 파일 읽기/쓰기 | cwd 기준 전체 파일 접근 | 코드 분석, 수정, 생성 |
| 셸 실행 | 모든 셸 명령 (sandbox 범위 내) | 빌드, 테스트, lint |
| **셸 병렬** | `& + wait`로 동시 실행 (5파일 2.3x, 20파일 2.8x) | 다수 파일 분석/검증 |
| 웹 검색 | 실시간 웹 검색 내장 | 최신 API 문서, 라이브러리 사용법 |
| 멀티턴 대화 | MCP 전용: `codex-reply` + threadId | 후속 질문, 점진적 수정 |
| `base-instructions` | MCP 전용: 기본 지시 오버라이드 | 프로젝트별 규칙 주입 |
| `developer-instructions` | MCP 전용: 개발자 역할 메시지 주입 | 프레임워크/컨벤션 주입 |

## 모드별 호출 패턴

### MCP 모드

```
# 첫 호출
mcp__codex__codex(
  prompt: "영어로 작성한 프롬프트",
  model: "gpt-5.4",
  config: {"reasoningEffort": "high"},
  approval-policy: "never",
  sandbox: "danger-full-access"
)

# 이어서 (멀티턴)
mcp__codex__codex-reply(
  thread_id: "이전 응답의 threadId",
  prompt: "후속 질문 또는 추가 지시"
)
```

### CLI 모드

```bash
# 기본 호출
codex exec \
  --model "gpt-5.4" \
  --reasoning-effort "high" \
  --approval-policy "never" \
  --sandbox "danger-full-access" \
  "영어로 작성한 프롬프트"

# 파일 내용 포함 시 (stdin 파이프)
cat target_file.py | codex exec \
  --model "gpt-5.4" \
  "Review this code for bugs. Focus on: error handling, type safety"

# 경량 호출 (같은 모델, reasoningEffort만 낮춤)
codex exec \
  --model "gpt-5.4" \
  --reasoning-effort "medium" \
  "Quick question: what does asyncio.gather return?"
```

> **CLI 모드 제약:** 멀티턴 불가. 매 호출이 새 세션. 컨텍스트를 이어가려면 이전 결과를 프롬프트에 직접 포함해야 한다.

## 프롬프트 규칙

- **Codex 프롬프트는 영어로** (토큰 ~30% 절약, 정확도 향상)
- **사용자 응답은 한국어로 정리**
- **프로젝트 구조는 Codex가 직접 파악하게 하라** — 특정 프로젝트 구조를 가정하지 말 것
- **검증 명령은 프로젝트 CLAUDE.md 참고** — 프로젝트마다 다르므로 하드코딩 금지
- **CLI 모드에서 긴 프롬프트:** 임시 파일에 쓰고 `codex exec < prompt.txt` 패턴 사용. 셸 이스케이핑 지옥을 피하라.

## 지뢰밭 (실전에서 데인 것들)

### 공통 (MCP + CLI)

| 함정 | 증상 | 대응 |
|------|------|------|
| 파일 범위 미지정 | Codex가 요청 안 한 파일까지 건드림 | 수정 대상 파일 목록 명시적 제한 |
| 한글 파일 수정 | 인코딩 깨짐 가능 | 수정 후 반드시 내용 검증 |
| 같은 파일 병렬 수정 | 충돌/덮어쓰기 | 파일 소유권 배타적 배분 |
| Codex 결과 맹신 | 오탐, 라인 번호 틀림 | Critical/High는 Read로 교차 검증 |
| 수정 전 파일 안 읽음 | 컨텍스트 없이 추측 수정 | "Read the file first" 프롬프트에 포함 |
| 에러 절대 수치로 판단 | 기존 에러를 신규로 오판 | 수정 전 베이스라인 잡고 델타로 판단 |

### MCP 전용

| 함정 | 증상 | 대응 |
|------|------|------|
| fix 후 threadId 재사용 | stale 컨텍스트로 엉뚱한 수정 | 코드 변경 후엔 항상 새 thread |

### CLI 전용

| 함정 | 증상 | 대응 |
|------|------|------|
| 셸 특수문자 이스케이핑 | 프롬프트에 `$`, `` ` ``, `"` 등이 있으면 깨짐 | 임시 파일 + stdin 리다이렉트 |
| 긴 stdout 잘림 | 출력이 너무 길면 Claude 컨텍스트 소모 | `--max-tokens` 제한 또는 출력을 파일로 리다이렉트 후 핵심만 추출 |
| exit code만 보고 판단 | exit 0이어도 내용이 엉뚱할 수 있음 | stdout 내용까지 반드시 확인 |
| 프로세스 hang | Codex가 응답 안 하면 Bash가 영원히 대기 | `timeout 180 codex exec ...` 감싸기 |

## 모드 선택 가이드

| 상황 | 추천 모드 | 이유 |
|------|-----------|------|
| 점진적 수정 (fix → 확인 → 추가 fix) | MCP | 멀티턴으로 컨텍스트 유지 |
| 프로젝트 규칙 주입 필요 | MCP | `base-instructions` 파라미터 |
| 단발성 코드 리뷰 | 둘 다 OK | 한 번 호출로 끝 |
| 대규모 병렬 분석 | CLI | timeout/kill 제어, 파이프 조합 |
| MCP 서버 연결 불안정 | CLI | 폴백으로 동작 보장 |
