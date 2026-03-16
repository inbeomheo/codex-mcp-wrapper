# Codex Hybrid Wrapper

Claude Code에서 OpenAI Codex를 활용하는 커맨드 파일. **MCP + CLI 하이브리드** 지원.

**파일 1개**(`codex.md`)를 설치하면 Claude Code가 Codex를 버그 분석, 코드 리뷰, 리팩토링, 구현, 자유 질문 등에 최적으로 활용합니다. MCP 연결 시 멀티턴·구조화 파라미터를 활용하고, MCP 미연결 시 CLI로 자동 폴백합니다.

## 왜 커맨드인가

Claude Code에 ChatGPT 계정을 연결하면 Codex MCP가 자동으로 추가됩니다. 별도 MCP 서버 설치가 필요 없습니다.

하지만 그냥 호출하면 최적 활용이 어렵습니다. 이 커맨드는 **3가지 원칙**, **모드 자동 감지**, **실전 지뢰밭**을 Claude에게 주입하여 Codex를 제대로 활용하게 합니다.

## 모드 자동 감지

| 조건 | 모드 | 특징 |
|------|------|------|
| `mcp__codex__codex` 도구 사용 가능 | MCP | 멀티턴, 구조화 파라미터, base-instructions |
| `command -v codex` 성공 | CLI | 단발성, Bash 실행, stdout 파싱 |
| 둘 다 불가 | 중단 | 설치 안내 |

## 3원칙

| # | 원칙 | 설명 |
|---|------|------|
| 1 | **Claude는 지휘, Codex는 실행** | Claude가 판단하고, Codex가 코드를 읽고/쓰고/실행한다 |
| 2 | **병렬은 Codex 내부에서** | MCP/CLI 모두 직렬 처리됨. 셸 병렬(`& + wait`)로 Codex 내부에서 병렬화 |
| 3 | **Codex 능력을 최대한 활용** | 단순 코드 생성이 아닌 셸 실행, 웹 검색, 멀티턴 등 전체 능력 위임 |

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
| 긴 stdout 잘림 | 출력이 너무 길면 Claude 컨텍스트 소모 | `--max-tokens` 제한 또는 파일 리다이렉트 |
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

## 설치

### 1. Codex CLI 설치 + 로그인

```bash
npm install -g @openai/codex
codex login
```

### 2. Codex MCP 서버 등록 (선택 — MCP 모드 사용 시)

```bash
claude mcp add codex -- codex mcp-server
```

Claude Code 재시작 후 Codex MCP 활성화. MCP 미등록 시 CLI 모드로 자동 폴백됩니다.

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

모드를 지정할 필요 없습니다. MCP/CLI를 자동 감지하고, Claude가 요청에 맞게 알아서 판단합니다.

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
model: "gpt-5.4-codex"        # 기본 (고성능)
model: "gpt-5.4-codex-spark"  # 경량 (빠른 응답)

config: {"reasoningEffort": "high"}   # 기본
config: {"reasoningEffort": "medium"} # 빠른 분석

sandbox: "danger-full-access"  # 기본 (전체 접근)
sandbox: "workspace-write"     # 쓰기 제한
```

## 버전 히스토리

- **v1**: 5개 모드(analyze, review, refactor, fix, question) 하드코딩, 273줄
- **v2**: 3원칙 + 능력표 + 지뢰밭, MCP 전용, 55줄
- **v3 (현재)**: MCP + CLI 하이브리드, 모드 자동 감지, CLI 지뢰밭 추가, 모드 선택 가이드

## 라이선스

MIT
