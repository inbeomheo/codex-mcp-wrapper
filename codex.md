---
name: codex
description: "Codex CLI-First 래퍼. 버그분석, 코드리뷰, 리팩토링, 구현, 질문, 병렬 평가 등 다목적 활용. CLI 병렬이 MCP 대비 2~7배 빠름. codex로 분석해줘, codex 리뷰해줘, codex 병렬로 돌려줘 등의 요청에 사용."
---

# Codex CLI-First Wrapper

## 핵심: CLI 우선, MCP는 선택

벤치마크 결과 CLI가 거의 모든 시나리오에서 MCP보다 우월하다.

| 시나리오 | CLI 병렬 | MCP 직렬 | 배율 |
|---------|---------|---------|------|
| 평가 3개 동시 | 14초 | ~30초 | **2x** |
| 경량 호출 5개 | 6초 | ~35초 | **6x** |
| 경량 호출 10개 | 10초 | ~70초 | **7x** |
| 멀티턴 대화 | `exec resume` | `codex-reply` | **동등** |
| 파일 읽기/쓰기 | ✅ 직접 | ❌ 불가 | **CLI only** |
| diff 적용 | `codex apply` | ❌ 불가 | **CLI only** |

## 모드 감지 (매 호출 시 자동 판별)

```
IF `command -v codex` 성공:
  → CLI 모드 (기본 — 병렬, 파일 접근, 멀티턴 모두 지원)
ELSE IF mcp__codex__codex 도구가 사용 가능:
  → MCP 모드 (폴백 — base-instructions 등 구조화 파라미터 필요 시)
ELSE:
  → "Codex가 설치되어 있지 않습니다" 안내 후 중단
```

## 기본 설정

`~/.codex/config.toml`에서 자동 적용:
```
model: "gpt-5.4"
reasoning_effort: "high"
sandbox: "danger-full-access"
```

## 3원칙

1. **Claude는 지휘, Codex는 실행.** Claude가 판단, Codex가 코드를 읽고/쓰고/실행.
2. **병렬은 CLI `& + wait`로.** MCP는 직렬 큐. CLI는 진짜 OS-level 병렬. N개 동시 실행 = 1개 시간.
3. **용도에 맞는 서브커맨드.** `exec`(범용), `review`(리뷰), `exec resume`(멀티턴), `apply`(diff 적용).

## Codex 능력

| 능력 | CLI | MCP | 활용 예 |
|------|-----|-----|---------|
| 파일 읽기/쓰기 | ✅ | ❌ | 코드 분석, 수정, 생성 |
| 셸 실행 | ✅ | ✅ (내부) | 빌드, 테스트, lint |
| **외부 병렬** | ✅ `& + wait` | ❌ 직렬 큐 | N개 파일 동시 분석 |
| 내부 병렬 | ✅ 셸 병렬 | ✅ 셸 병렬 | Codex 안에서 `& + wait` |
| 웹 검색 | ✅ | ✅ | 최신 API 문서 |
| 멀티턴 대화 | ✅ `exec resume` | ✅ `codex-reply` | 후속 질문, 점진적 수정 |
| `base-instructions` | ⚠️ config.toml | ✅ 파라미터 | 프로젝트별 규칙 주입 |
| diff 적용 | ✅ `codex apply` | ❌ | 마지막 수정사항 적용 |
| 코드 리뷰 | ✅ `codex review` | ❌ | 전용 리뷰 서브커맨드 |

---

## CLI 호출 패턴

### 1. 단일 작업

```bash
codex exec -o /tmp/result.txt "프롬프트 내용"
```

주요 옵션:
- `-o <파일>`: 출력을 파일로 저장 (파싱 용이)
- `-s read-only`: 읽기 전용 sandbox
- `-m <모델>`: 모델 지정
- `-c 'model_reasoning_effort="xhigh"'`: 추론 강도 오버라이드

### 2. 병렬 실행 (핵심 패턴)

```bash
codex exec -o /tmp/r1.txt "파일 A 분석" &
codex exec -o /tmp/r2.txt "파일 B 분석" &
codex exec -o /tmp/r3.txt "파일 C 분석" &
wait
# 결과 수집
for f in /tmp/r{1,2,3}.txt; do cat "$f"; done
```

대규모 병렬 (autoresearch, 다중 평가 등):
```bash
for i in $(seq 1 10); do
  codex exec -o "/tmp/eval_${i}.txt" "Evaluate prompt ${i}" &
done
wait
```

### 3. 코드 리뷰

```bash
codex review --base main
# 커스텀 지시 추가:
codex review "보안 중심으로" --base main
```

### 4. 멀티턴 대화

```bash
# 첫 호출 — 세션 ID가 출력에 포함됨
codex exec "이 프로젝트의 아키텍처를 분석해줘"
# → session id: 019d1158-28d8-...

# 이어서 질문
codex exec resume 019d1158-28d8-... "그럼 성능 병목은 어디야?"

# 가장 최근 세션 이어받기
codex exec resume --last "추가 질문"
```

### 5. diff 적용

```bash
codex apply  # 마지막 Codex 수정사항을 git apply
```

### 6. Challenge (Adversarial)

```bash
codex exec -s read-only \
  -c 'model_reasoning_effort="xhigh"' \
  "Review git diff origin/main. Find ways this code will fail in production.
   Edge cases, race conditions, security holes. Be adversarial."
```

---

## MCP 모드 (폴백/특수 용도)

CLI가 없거나, `base-instructions` 파라미터가 필요한 경우에만 사용:

```
mcp__codex__codex(
  prompt: "프롬프트",
  base-instructions: "프로젝트 규칙...",
  model: "gpt-5.4",
  config: {"reasoningEffort": "high"},
  sandbox: "danger-full-access"
)
```

**주의:** MCP는 직렬 큐. 여러 호출이 동시에 들어와도 순차 처리됨.

---

## 프롬프트 규칙

- **Codex 프롬프트는 영어로** (토큰 ~30% 절약)
- **사용자 응답은 한국어로 정리**
- **파일 범위 명시** — 수정 대상 파일 목록을 제한
- **"Read the file first"** — 수정 전 파일 읽기 지시 포함
- **긴 프롬프트:** heredoc 또는 임시 파일 사용

```bash
codex exec -o /tmp/out.txt "$(cat <<'EOF'
여기에 긴 프롬프트...
여러 줄도 가능...
EOF
)"
```

## reasoning_effort 가이드

| 용도 | 설정 | 이유 |
|------|------|------|
| 코드 리뷰 | `high` | 깊이 있되 빠르게 |
| Adversarial challenge | `xhigh` | 최대 추론력 |
| 일반 질문/분석 | `high` | 균형 |
| 반복 평가 (autoresearch) | `high` | 비용/속도 고려 |
| 경량 확인 | `medium` | 빠른 응답 |

---

## 지뢰밭

| 함정 | 증상 | 대응 |
|------|------|------|
| 파일 범위 미지정 | 엉뚱한 파일까지 수정 | 수정 대상 파일 목록 명시 |
| 한글 파일 수정 | 인코딩 깨짐 | 수정 후 내용 검증 |
| 같은 파일 병렬 수정 | 충돌/덮어쓰기 | 파일 소유권 배타적 배분 |
| Codex 결과 맹신 | 오탐, 틀린 라인 | Critical은 Read로 교차 검증 |
| 셸 특수문자 | `$`, `` ` `` 포함 프롬프트 깨짐 | heredoc 또는 임시 파일 |
| 프로세스 hang | 무한 대기 | `timeout 180 codex exec ...` |
| resume 실패 | 세션 만료/삭제 | 새 세션으로 시작 |
| MCP 직렬 큐 | N개 호출이 N배 시간 | CLI `& + wait` 병렬 사용 |
