# Codex MCP Universal Wrapper

Claude Code에서 OpenAI Codex MCP를 최적 설정으로 호출하는 범용 커맨드.

별도 MCP 서버 설치 없이, **Claude Code 커맨드 하나로** Codex를 버그 분석, 코드 리뷰, 리팩토링, 버그 수정, 자유 질문에 활용합니다.

## vs codex-mcp-bridge

| 항목 | 이 프로젝트 | codex-mcp-bridge |
|------|-----------|-----------------|
| **구조** | Claude Code 커맨드 1개 | 독립 stdio MCP 서버 |
| **설치** | 파일 1개 복사 | `pip install` + `claude mcp add` |
| **모델 제어** | 직접 지정 (`gpt-5.3-codex`) | CLI가 자동 선택 |
| **컨텍스트 유지** | threadId 캐시로 후속 대화 | 매번 새 세션 |
| **커맨더 자율성** | Claude가 상황 판단해서 유연 대응 | 8개 스킬로 고정 분리 |
| **실전 교훈** | 63건 버그 수정 경험 내장 | 없음 |
| **레이어** | MCP 직접 호출 | CLI → MCP (중간 레이어 +1) |

## 주요 기능

### 5가지 모드

| 모드 | 사용법 | 설명 |
|------|--------|------|
| 버그 분석 | `/codex analyze` | 병렬 서브에이전트로 대규모 분석, 자동 검증 + 중복 제거 |
| 코드 리뷰 | `/codex review` | git diff 기반 리뷰, threadId로 후속 질문 |
| 리팩토링 | `/codex refactor` | before/after 코드 비교, 사용자 승인 후 적용 |
| 버그 수정 | `/codex fix` | 에러 베이스라인 → 배치 수정 → 자동 검증 |
| 자유 질문 | `/codex {질문}` | 기술 질문, 설계 상담 등 |

### 핵심 특징

- **자동 검증**: Critical/High 이슈를 실제 코드와 교차 검증, 오탐 자동 제거
- **중복 제거**: 여러 에이전트 결과를 file:line 기준으로 자동 dedup
- **에러 베이스라인**: 수정 전후 `tsc` 에러 수를 델타로 비교
- **병렬 실행**: 3개+ 파일은 서브에이전트 병렬 분석 (최대 5개 동시)
- **파일 소유권**: 에이전트별 배타적 파일 배분으로 충돌 방지
- **threadId 캐시**: 후속 질문에서 컨텍스트 유지 (토큰 절약)

### 실전에서 검증된 교훈

63건 버그를 5-agent 병렬로 분석/수정한 경험에서 얻은 규칙:

- 검증은 `npx tsc --noEmit` 우선, `npm run build`는 보조
- fix 에이전트는 반드시 파일을 먼저 읽고 수정
- fix 모드에서 analyze threadId 재사용 금지 (stale 컨텍스트)
- 핵심 타입 1개 수정으로 28개 에러 해결 가능 (전파 인지)
- `src/` 디렉토리 가정 금지 — 항상 실제 구조 확인

## 설치

### 방법 1: 파일 복사 (권장)

```bash
# Claude Code 커맨드 디렉토리에 복사
cp codex.md ~/.claude/commands/codex.md
```

### 방법 2: 프로젝트별 설치

```bash
# 프로젝트 루트에 복사
mkdir -p .claude/commands
cp codex.md .claude/commands/codex.md
```

## 사전 요구사항

- [Claude Code](https://claude.ai/code) 설치
- [Codex MCP](https://chatgpt.com/codex) 연결 (ChatGPT 계정 필요)
- Claude Code에서 Codex MCP 서버가 활성화되어 있어야 함

## 사용법

Claude Code에서:

```
/codex 이 프로젝트 버그 분석해줘
/codex src/config.py 리뷰해줘
/codex useAuth 훅 리팩토링 제안해줘
/codex 검증된 버그 리스트 기반으로 수정해줘
/codex MCP stdio 전송 방식이 뭐야?
```

## 설정 커스터마이징

`codex.md` 파일에서 기본 설정을 수정할 수 있습니다:

```yaml
# 모델 변경
model: "gpt-5.3-codex"        # 기본 (고성능)
model: "gpt-5.3-codex-spark"  # 경량 (빠른 응답)

# 추론 강도
config: {"reasoningEffort": "high"}   # 기본
config: {"reasoningEffort": "medium"} # 빠른 분석

# 샌드박스
sandbox: "danger-full-access"    # 기본 (전체 접근)
sandbox: "workspace-write"       # 쓰기 제한
```

## 라이선스

MIT License
