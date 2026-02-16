---
name: codex
description: Codex MCP 범용 래퍼. 버그분석, 코드리뷰, 리팩토링, 질문 등 다목적 Codex 활용. "codex로 분석해줘", "codex 버그 찾아줘", "codex 리뷰해줘" 등의 요청에 사용.
---

# Codex MCP Universal Wrapper

Codex MCP를 최적 설정으로 호출하는 범용 스킬.

## 기본 설정 (모든 호출에 적용)

```
model: "gpt-5.3-codex"
config: {"reasoningEffort": "high"}
approval-policy: "never"
sandbox: "danger-full-access"
```

경량 작업(단순 질문, 짧은 확인)은 spark 모델 사용:
```
model: "gpt-5.3-codex-spark"
```

## 핵심 규칙

1. **Codex 프롬프트는 반드시 영어로 작성** (토큰 ~30% 절약, 정확도 향상)
2. **사용자 응답은 한국어로 정리**
3. **threadId 캐시**: 같은 파일 후속 질문은 `codex-reply` 사용 (단, fix 모드에서는 재사용 금지)
4. **자동 검증**: Critical/High 이슈는 Read 도구로 자동 교차 검증 (수동 확인 불요)
5. **중복 제거**: 여러 에이전트 결과 병합 시 동일 file:line 이슈 자동 dedup

## 실전 교훈 (Lessons Learned)

### 검증은 `tsc` 우선, `build`는 보조
- **1차 검증**: `npx tsc --noEmit` — 순수 타입 체크, 환경 무관
- **2차 검증**: `npm run build` — PostCSS/bundler 등 환경 이슈로 실패 가능
- **3차 검증**: `npm test -- --run` — 로직 검증
- `npm run build` 실패 시 tsc 통과했으면 환경 이슈로 판단, 수정 코드 탓 아님

### 에러 베이스라인 먼저 잡기
- 수정 시작 전 `npx tsc --noEmit 2>&1 | wc -l`로 기존 에러 수 기록
- 매 배치 후 동일 명령으로 "새 에러 = 0"인지 확인
- 절대 수치가 아닌 **델타(증감)**로 판단

### 에이전트 프롬프트에 프로젝트 구조 포함 필수
```
Project structure (IMPORTANT):
- Source files are at ROOT level, NOT under src/
- cwd: {actual_cwd_path}
- Key directories: components/, hooks/, services/, utils/, types/
- DO NOT assume src/ directory exists
```

### fix 에이전트는 반드시 파일을 먼저 읽어야 함
- 분석 결과의 설명만 보고 수정하면 빗나감
- fix 프롬프트에 반드시 포함: "Read the file first before making any changes"

### 파일 소유권 배타적 배분
- 같은 파일을 여러 에이전트가 수정하면 충돌 발생
- 에이전트별 파일 목록을 겹치지 않게 분배
- 공유 파일(types/, utils/)은 단 1개 에이전트만 담당

### fix 모드에서 threadId 재사용 금지
- analyze threadId를 fix에서 재사용하면 stale 컨텍스트 문제
- 코드가 변경된 후에는 항상 새 thread로 시작

### 타입 변경 전파 인지
- 핵심 타입 1개 수정 → 연쇄적으로 여러 파일 에러 해결되기도 함
- 반대로, 타입 변경이 소비자(consumer) 파일에 새 에러를 만들 수 있음
- 타입 수정 후 즉시 `npx tsc --noEmit`으로 전파 영향 확인

## 모드별 워크플로우

### 모드 1: 버그 분석 (`/codex analyze` 또는 `버그 찾아줘`)

1. **프로젝트 구조 파악** (Glob으로 주요 파일 확인)
   - `src/` 가정 금지! 실제 파일 위치 확인 후 진행
2. **에러 베이스라인 기록**: `npx tsc --noEmit 2>&1 | grep "error TS" | wc -l`
3. **파일 분담 전략**: 파일 간 겹침 최소화
   - 각 에이전트에 명확한 파일 목록 할당
   - 공유 유틸/타입 파일은 1개 에이전트만 담당
4. Task 서브에이전트 병렬로 Codex 투입 (파일/영역별 분담)
   - 각 서브에이전트가 Codex MCP 호출
   - 프롬프트에 반드시 포함:
   ```
   Project cwd: {cwd_path}
   Source files are at ROOT level, NOT under src/.

   Analyze the following files for bugs, race conditions, memory leaks, and logic errors:
   {file_list}

   For each issue report: file:line, severity (Critical/High/Medium/Low),
   description, and fix suggestion. Be concise.
   Output as JSON array: [{"file":"...","line":N,"severity":"...","desc":"...","fix":"..."}]
   ```
5. **자동 중복 제거**: 결과 병합 시 동일 file:line 항목 통합
   - 동일 위치 + 유사 설명 → 1건으로 병합 (더 높은 severity 채택)
   - 겹치는 라인 범위 (±3줄) → 대표 1건만 유지
6. **자동 교차 검증** (Critical/High만):
   - Read 도구로 해당 file:line 읽기
   - 실제 코드와 Codex 보고 내용 비교
   - 일치 → "검증됨", 불일치 → "오탐" 태그
7. 검증된 버그만 최종 리포트로 제시

### 모드 2: 코드 리뷰 (`/codex review`)

1. 변경된 파일 확인 (git diff 또는 사용자 지정)
2. Codex에 리뷰 요청:
   ```
   Review this code for: correctness, performance, security, readability.
   Flag only real issues with file:line and severity. No nitpicks.
   ```
3. threadId 캐시 → 후속 질문으로 세부 확인

### 모드 3: 리팩토링 제안 (`/codex refactor`)

1. 대상 파일/함수 Codex에 전달
2. 리팩토링 제안 요청:
   ```
   Suggest refactoring for {file/function}. Focus on:
   reducing complexity, improving maintainability, eliminating duplication.
   Show before/after code snippets.
   ```
3. 사용자 승인 후 적용

### 모드 4: 버그 수정 (`/codex fix`)

1. **입력**: 검증된 버그 리스트 (analyze 결과 또는 사용자 제공)
2. **에러 베이스라인 기록**: `npx tsc --noEmit` 기존 에러 수 저장
3. **수정 전략 수립**:
   - 버그를 파일별로 그룹핑
   - 의존관계 분석 (A 수정이 B에 영향?)
   - 독립적인 그룹은 병렬, 의존적인 그룹은 순차
   - **파일 소유권 배타적 배분** — 같은 파일 2개 에이전트 금지
4. **Codex에 수정 요청** (새 thread — analyze threadId 재사용 금지):
   ```
   Project cwd: {cwd_path}
   Source files are at ROOT level, NOT under src/.

   IMPORTANT: Read each file FIRST before making changes.

   Fix the following verified bugs:
   - {file}:{line}: {description}
   ...

   Rules:
   - Minimal changes only. Do not refactor unrelated code.
   - Do not add comments, docstrings, or type annotations to unchanged code.
   - Preserve existing behavior for non-buggy code paths.
   - After fixing, verify with: npx tsc --noEmit
   ```
5. **배치 후 검증** (매 배치마다):
   - `npx tsc --noEmit` — 새 에러 0건 확인 (베이스라인 대비)
   - `npm test -- --run` — 기존 테스트 통과
   - `npm run build` — 빌드 가능하면 보너스 (실패해도 tsc 통과면 OK)
6. **실패 시 롤백**: git checkout으로 복원, 사용자에게 보고

### 모드 5: 자유 질문 (`/codex {질문}`)

1. 사용자 질문을 영어로 변환하여 Codex에 전달
2. 결과를 한국어로 정리하여 반환
3. threadId 보관 → 후속 대화 가능

## 병렬 실행 전략

| 상황 | 방법 |
|------|------|
| 파일 1-2개 분석 | 직접 MCP 호출 |
| 파일 3개+ 분석 | Task 서브에이전트 병렬 (각각 Codex MCP 호출) |
| 후속 질문 | codex-reply(threadId) — 토큰 절약 |
| 버그 수정 | 파일별 그룹 → 독립 그룹은 병렬 서브에이전트 (새 thread) |

## 서브에이전트 호출 템플릿

```
Task 서브에이전트 호출 시:
- subagent_type: "general-purpose"
- prompt에 포함할 내용:
  1. mcp__codex__codex 도구 호출 지시
  2. 영어 프롬프트 (프로젝트 구조 + cwd 포함)
  3. model/config/cwd/approval-policy/sandbox 설정
  4. "결과를 그대로 반환해줘"
- 최대 5개 동시 (API 부하 방지)
- 각 에이전트에 배타적 파일 목록 배분
```

## 자동 검증 절차 (analyze 모드)

Critical/High 이슈 발견 시 자동 실행:

```
1. Read(file, offset=line-5, limit=15) → 해당 코드 확인
2. Codex 보고 내용과 실제 코드 비교:
   - 보고된 변수/함수명이 실제 존재하는가?
   - 보고된 로직 흐름이 실제와 일치하는가?
   - 보고된 라인 번호가 정확한가? (±3줄 허용)
3. 판정:
   - ✅ 검증됨: 실제 코드에서 문제 확인
   - ⚠️ 부분 일치: 라인 번호 오차 있으나 이슈는 실재
   - ❌ 오탐: 실제 코드와 불일치
4. 오탐은 최종 리포트에서 제외
```

## 중복 제거 알고리즘

여러 에이전트가 같은 파일을 참조할 때:

```
1. file:line 기준 정렬
2. 동일 file + |line1 - line2| <= 3 → 후보 그룹
3. 후보 그룹 내 설명 유사도 확인 (같은 변수/함수 언급)
4. 유사 → 병합 (더 높은 severity, 더 상세한 설명 채택)
5. 비유사 → 별도 이슈로 유지
```

## 출력 포맷

### 버그 리포트
```markdown
## Codex 분석 결과

### Critical (N건) — 자동 검증 완료
| 위치 | 설명 | 수정 제안 | 검증 |
|------|------|-----------|------|
| file:line | 문제 설명 | 해결 방법 | ✅/⚠️ |

### High (N건) — 자동 검증 완료
...

### Medium (N건)
| 위치 | 설명 | 수정 제안 |
|------|------|-----------|
| file:line | 문제 설명 | 해결 방법 |

### Low (N건)
...

### 검증 요약
- 총 발견: N건 → 중복 제거 후: M건
- 자동 검증: Critical K건, High J건
- 오탐 제거: X건
- 최종 확정: Y건
- 에러 베이스라인: X건 (수정 전) → Y건 (수정 후)
```

### 수정 리포트 (fix 모드)
```markdown
## Codex 수정 결과

### 에러 베이스라인
- 수정 전: N건 → 수정 후: M건 (신규 에러: 0건)

### 수정 완료 (N건)
| 위치 | 문제 | 수정 내용 | tsc | 테스트 |
|------|------|----------|-----|--------|
| file:line | 설명 | 변경 요약 | ✅/❌ | ✅/❌ |

### 수정 실패 (N건)
| 위치 | 문제 | 실패 사유 |
|------|------|----------|
| file:line | 설명 | 이유 |
```

## 주의사항

- Codex 결과를 100% 신뢰하지 말 것 — Critical/High는 자동 검증으로 필터링
- 코드 수정은 사용자 승인 후에만 진행 (fix 모드에서도 diff 보여주고 확인)
- 대규모 분석 시 서브에이전트 5개 이하로 제한 (API 부하 방지)
- fix 모드 실패 시 즉시 롤백, 절대 깨진 상태로 두지 않음
- 수정 후 `npx tsc --noEmit` 통과가 최소 기준, 빌드+테스트는 추가 검증
- `src/` 디렉토리 가정 금지 — 항상 실제 구조 확인 후 진행
- fix 모드에서 analyze threadId 재사용 금지 — 항상 새 thread
