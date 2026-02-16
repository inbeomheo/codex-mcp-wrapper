# Codex MCP Universal Wrapper

Claude Code에서 OpenAI Codex MCP를 최적 설정으로 호출하는 범용 커맨드.

별도 MCP 서버 설치 없이, **Claude Code 커맨드 파일 하나**로 Codex를 버그 분석, 코드 리뷰, 리팩토링, 버그 수정, 자유 질문에 활용합니다.

## 왜 MCP 서버가 아니라 커맨드인가

기존 Codex MCP 프로젝트들은 Codex CLI를 MCP 서버로 감싸는 방식입니다. Claude Code에 ChatGPT 계정을 연결하면 Codex MCP가 자동으로 추가되기 때문에, 별도 MCP 서버를 설치할 필요가 없습니다.

이 프로젝트는 **커맨드 파일 1개**로 Codex MCP를 직접 호출합니다. 설치할 것도, 실행할 서버도 없습니다.

## 다른 프로젝트에 없는 기능

63건 버그를 5-agent 병렬로 분석/수정한 실전 경험에서 만들어진 기능들입니다.

### 자동 검증 + 오탐 제거

Codex가 보고한 Critical/High 이슈를 실제 코드와 자동 교차 검증합니다.

```
1. Read(file, offset=line-5, limit=15) → 해당 코드 확인
2. 보고된 변수/함수명이 실제 존재하는지, 라인 번호가 맞는지 비교
3. 검증됨 / 부분 일치 / 오탐 판정
4. 오탐은 최종 리포트에서 자동 제거
```

### 중복 제거 알고리즘

여러 에이전트가 같은 버그를 다른 표현으로 보고할 때 자동 병합합니다.

```
1. file:line 기준 정렬
2. 동일 file + |line1 - line2| <= 3 → 후보 그룹
3. 같은 변수/함수 언급 → 1건으로 병합 (더 높은 severity 채택)
4. 비유사 → 별도 이슈로 유지
```

### 에러 베이스라인

수정 전후를 절대 수치가 아닌 **델타(증감)**로 판단합니다.

```bash
# 수정 전 기록
npx tsc --noEmit 2>&1 | grep "error TS" | wc -l  # → 35건

# 매 배치 후 확인: 새 에러 = 0이면 통과
npx tsc --noEmit 2>&1 | grep "error TS" | wc -l  # → 35건 이하면 OK
```

### 파일 소유권 배타적 배분

같은 파일을 여러 에이전트가 동시에 수정하면 충돌합니다. 에이전트별로 파일 목록을 겹치지 않게 분배하고, 공유 파일(types/, utils/)은 단 1개 에이전트만 담당합니다.

### threadId 캐시

후속 질문에서 컨텍스트를 유지합니다. 단, fix 모드에서는 코드가 변경된 후이므로 항상 새 thread로 시작합니다. (stale 컨텍스트 방지)

## 주요 기능

### 5가지 모드

| 모드 | 사용법 | 설명 |
|------|--------|------|
| 버그 분석 | `/codex analyze` | 병렬 서브에이전트로 대규모 분석, 자동 검증 + 중복 제거 |
| 코드 리뷰 | `/codex review` | git diff 기반 리뷰, threadId로 후속 질문 |
| 리팩토링 | `/codex refactor` | before/after 코드 비교, 사용자 승인 후 적용 |
| 버그 수정 | `/codex fix` | 에러 베이스라인 → 배치 수정 → 자동 검증 |
| 자유 질문 | `/codex {질문}` | 기술 질문, 설계 상담 등 |

### 커맨더 자율성

모드를 8개 스킬로 쪼개지 않습니다. Claude가 상황에 맞게 모델, 샌드박스, 병렬 수를 직접 판단합니다.

## 처음부터 세팅하기

### Step 1: Claude Code 설치

```bash
npm install -g @anthropic-ai/claude-code
```

설치 후 `claude` 명령으로 실행 확인.

### Step 2: ChatGPT 계정 연결 (Codex MCP 활성화)

Claude Code 실행 후:

```
/login
```

또는 Settings에서 ChatGPT 계정을 연결합니다. 연결하면 Codex MCP 서버가 자동으로 추가됩니다.

연결 확인:

```bash
claude mcp list
```

목록에 `codex` 또는 `chatgpt` 관련 MCP 서버가 보이면 성공입니다.

### Step 3: 커맨드 파일 설치

```bash
# 이 레포 클론
git clone https://github.com/inbeomheo/codex-mcp-wrapper.git
cd codex-mcp-wrapper

# 전역 설치 (권장 — 모든 프로젝트에서 사용 가능)
cp codex.md ~/.claude/commands/codex.md
```

프로젝트별로만 쓰려면:

```bash
mkdir -p .claude/commands
cp codex.md .claude/commands/codex.md
```

### Step 4: 동작 확인

Claude Code에서:

```
/codex 안녕, 연결 테스트야
```

Codex가 응답하면 세팅 완료입니다.

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
