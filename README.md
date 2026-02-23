# Codex MCP Wrapper

Claude Code에서 OpenAI Codex MCP를 활용하는 커맨드 파일.

**파일 1개**(`codex.md`)를 설치하면 Claude Code가 Codex를 버그 분석, 코드 리뷰, 리팩토링, 구현, 자유 질문 등에 최적으로 활용합니다.

## 왜 커맨드인가

Claude Code에 ChatGPT 계정을 연결하면 Codex MCP가 자동으로 추가됩니다. 별도 MCP 서버 설치가 필요 없습니다.

하지만 그냥 호출하면 최적 활용이 어렵습니다. 이 커맨드는 **3가지 원칙**과 **실전 지뢰밭**을 Claude에게 주입하여 Codex를 제대로 활용하게 합니다.

## 3원칙

| # | 원칙 | 설명 |
|---|------|------|
| 1 | **Claude는 지휘, Codex는 실행** | Claude가 판단하고, Codex가 코드를 읽고/쓰고/실행한다 |
| 2 | **병렬은 Codex 내부에서** | MCP 다중 호출은 직렬 처리됨. 셸 병렬(`& + wait`)로 Codex 내부에서 병렬화 |
| 3 | **Codex 능력을 최대한 활용** | 단순 코드 생성이 아닌 셸 실행, 웹 검색, 멀티턴 등 전체 능력 위임 |

## Codex 능력 (MCP 서브에이전트로 사용 가능한 것만)

| 능력 | 설명 | 활용 예 |
|------|------|---------|
| 파일 읽기/쓰기 | cwd 기준 전체 파일 접근 | 코드 분석, 수정, 생성 |
| 셸 실행 | 모든 셸 명령 (sandbox 범위 내) | 빌드, 테스트, lint |
| **셸 병렬** | `& + wait`로 동시 실행 (5파일 2.3x, 20파일 2.8x) | 다수 파일 분석/검증 |
| 웹 검색 | 실시간 웹 검색 내장 | 최신 API 문서, 라이브러리 사용법 |
| 멀티턴 대화 | `codex-reply` + threadId | 후속 질문, 점진적 수정 |
| `base-instructions` | 기본 지시 오버라이드 | 프로젝트별 규칙 주입 |
| `developer-instructions` | 개발자 역할 메시지 주입 | 프레임워크/컨벤션 주입 |

## 지뢰밭 (실전에서 데인 것들)

63건 버그를 병렬 에이전트로 분석/수정하면서 발견한 함정들입니다.

| 함정 | 증상 | 대응 |
|------|------|------|
| fix 후 threadId 재사용 | stale 컨텍스트로 엉뚱한 수정 | 코드 변경 후엔 항상 새 thread |
| 파일 범위 미지정 | Codex가 요청 안 한 파일까지 건드림 | 수정 대상 파일 목록 명시적 제한 |
| 한글 파일 수정 | 인코딩 깨짐 가능 | 수정 후 반드시 내용 검증 |
| 같은 파일 병렬 수정 | 충돌/덮어쓰기 | 파일 소유권 배타적 배분 |
| Codex 결과 맹신 | 오탐, 라인 번호 틀림 | Critical/High는 Read로 교차 검증 |
| 수정 전 파일 안 읽음 | 컨텍스트 없이 추측 수정 | "Read the file first" 프롬프트에 포함 |
| 에러 절대 수치로 판단 | 기존 에러를 신규로 오판 | 수정 전 베이스라인 잡고 델타로 판단 |

## 설치

### 1. Codex CLI 설치 + 로그인

```bash
npm install -g @openai/codex
codex login
```

### 2. Codex MCP 서버 등록

```bash
claude mcp add codex -- codex mcp-server
```

Claude Code 재시작 후 Codex MCP 활성화.

### 3. 커맨드 파일 설치

```bash
# 전역 (모든 프로젝트에서 사용)
cp codex.md ~/.claude/commands/codex.md

# 프로젝트별
mkdir -p .claude/commands
cp codex.md .claude/commands/codex.md
```

### 4. 확인

```
/codex 연결 테스트
```

## 사용법

모드를 지정할 필요 없습니다. Claude가 요청에 맞게 알아서 판단합니다.

```
/codex 이 프로젝트 버그 분석해줘
/codex config.py 리뷰해줘
/codex useAuth 훅 리팩토링 제안해줘
/codex 검증된 버그 수정해줘
/codex MCP stdio 전송 방식이 뭐야?
```

## 설정 커스터마이징

`codex.md`에서 기본 설정 변경:

```yaml
model: "gpt-5.3-codex"        # 기본 (고성능)
model: "gpt-5.3-codex-spark"  # 경량 (빠른 응답)

config: {"reasoningEffort": "high"}   # 기본
config: {"reasoningEffort": "medium"} # 빠른 분석

sandbox: "danger-full-access"  # 기본 (전체 접근)
sandbox: "workspace-write"     # 쓰기 제한
```

## 이전 버전과의 차이

v1은 5개 모드(analyze, review, refactor, fix, question)를 하드코딩하고 273줄이었습니다.

v2는 **3원칙 + 능력표 + 지뢰밭** 55줄로, Claude가 상황에 맞게 유연하게 판단합니다.

## 라이선스

MIT
