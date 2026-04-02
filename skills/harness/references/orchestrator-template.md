# 오케스트레이터 스킬 템플릿

오케스트레이터는 전체 워크플로우를 조율하는 상위 스킬이다. 실행 모드에 따라 두 가지 템플릿을 제공한다.

---

## 템플릿 A: Task 병렬 호출 (기본)

Task를 run_in_background: true로 동시 호출하여 병렬 실행한다.

markdown
---
name: {domain}-orchestrator
description: "{도메인} 워크플로우를 조율하는 오케스트레이터. {트리거 키워드}."
---

# {Domain} Orchestrator

{도메인}의 Task를 조율하여 {최종 산출물}을 생성하는 통합 스킬.

## 실행 모드: Task 병렬 호출

## 에이전트 구성

| 에이전트 | 역할 | 스킬 | 출력 |
|---------|------|------|------|
| {agent-1} | {역할} | {skill} | _workspace/01_{agent-1}.md |
| {agent-2} | {역할} | {skill} | _workspace/02_{agent-2}.md |
| ... | | | |

## 워크플로우

### Phase 1: 준비
1. 사용자 입력 분석 — {무엇을 파악하는지}
2. 작업 디렉토리에 _workspace/ 생성
3. 입력 데이터를 _workspace/00_input/에 저장

### Phase 2: 병렬 Task 실행

모든 Task를 동시에 호출:

Task({ name: "agent-1", prompt: "{역할 설명 및 작업 지시}", model: "openrouter/minimax/minimax-m2.7", run_in_background: true })
Task({ name: "agent-2", prompt: "{역할 설명 및 작업 지시}", model: "openrouter/minimax/minimax-m2.7", run_in_background: true })
Task({ name: "agent-3", prompt: "{역할 설명 및 작업 지시}", model: "openrouter/minimax/minimax-m2.7", run_in_background: true })


모든 Task 완료 대기 → 결과 취합

**산출물 저장:**

| 에이전트 | 출력 경로 |
|---------|----------|
| {agent-1} | _workspace/01_{agent-1}_{artifact}.md |
| {agent-2} | _workspace/02_{agent-2}_{artifact}.md |

### Phase 3: 결과 취합 및 통합
1. 모든 Task 결과 Read
2. {통합/검증 로직}
3. 최종 산출물 생성: {output-path}/{filename}

### Phase 4: 정리
1. _workspace/ 디렉토리 보존 (중간 산출물은 삭제하지 않음 — 사후 검증·감사 추적용)
2. 사용자에게 결과 요약 보고

## 데이터 흐름


[parent]
    ├── Task({ name: "agent-1", run_in_background: true })
    ├── Task({ name: "agent-2", run_in_background: true })
    │
    ├── 결과 수신 ( artifact-1.md )
    └── 결과 수신 ( artifact-2.md )
              │
              └────── Read ───────┘
                          │
                    [parent: 통합]
                          │
                    최종 산출물


## 에러 핸들링

| 상황 | 전략 |
|------|------|
| Task 1개 실패 | 1회 재시도. 재실패 시 해당 결과 없이 진행, 보고서에 누락 명시 |
| Task 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| 타임아웃 | 현재까지 수집된 부분 결과 사용 |
| 데이터 충돌 | 출처 명시 후 병기, 삭제하지 않음 |

## 테스트 시나리오

### 정상 흐름
1. 사용자가 {입력}을 제공
2. Phase 1에서 {분석 결과} 도출
3. Phase 2에서 {N}개 Task 병렬 실행
4. Phase 3에서 산출물 통합하여 최종 결과 생성
5. 예상 결과: {output-path}/{filename} 생성

### 에러 흐름
1. Phase 2에서 {agent-2}가 실패
2. 1회 재시도 후에도 실패
3. {agent-2} 결과 없이 나머지 결과로 Phase 3 진행
4. 최종 보고서에 "{agent-2} 영역 미수집" 명시


---

## 템플릿 B: Task 순차 호출

이전 Task의 결과를 다음 Task의 입력으로 전달하며 순차 실행한다.

markdown
---
name: {domain}-orchestrator
description: "{도메인} 워크플로우를 조율하는 오케스트레이터. {트리거 키워드}."
---

# {Domain} Orchestrator

{도메인}의 Task를 순차적으로 조율하여 {최종 산출물}을 생성하는 통합 스킬.

## 실행 모드: Task 순차 호출

## 에이전트 구성

| 에이전트 | 역할 | 스킬 | 출력 |
|---------|------|------|------|
| {agent-1} | {역할} | {skill} | _workspace/01_{agent-1}.md |
| {agent-2} | {역할} | {skill} | _workspace/02_{agent-2}.md |
| ... | | | |

## 워크플로우

### Phase 1: 준비
1. 사용자 입력 분석 — {무엇을 파악하는지}
2. 작업 디렉토리에 _workspace/ 생성
3. 입력 데이터를 _workspace/00_input/에 저장

### Phase 2: 순차 Task 실행

이전 Task의 결과를 다음 Task 입력으로 전달:


Phase 2-1: Task({ name: "agent-1", prompt: "{역할 설명}", model: "openrouter/minimax/minimax-m2.7" })
→ 결과: _workspace/01_agent-1.md

Phase 2-2: Task({ name: "agent-2", prompt: "{역할 설명}. 입력: _workspace/01_agent-1.md 참고", model: "openrouter/minimax/minimax-m2.7" })
→ 결과: _workspace/02_agent-2.md

Phase 2-3: Task({ name: "agent-3", prompt: "{역할 설명}. 입력: _workspace/02_agent-2.md 참고", model: "openrouter/minimax/minimax-m2.7" })
→ 결과: _workspace/03_agent-3.md


### Phase 3: 최종 통합
1. 최종 Task 결과 Read
2. {최종 통합/검증 로직}
3. 최종 산출물 생성: {output-path}/{filename}

### Phase 4: 정리
1. _workspace/ 디렉토리 보존
2. 사용자에게 결과 요약 보고

## 데이터 흐름


[parent]
    │
    ├── Task({ name: "agent-1" }) → artifact-1.md
    │                                    │
    │                                    ↓
    ├── Task({ name: "agent-2" }) ← artifact-1.md 입력
    │                                    │
    │                                    ↓
    ├── Task({ name: "agent-3" }) ← artifact-2.md 입력
    │                                    │
    └────────────────────────────────────┘
                        │
                  [parent: 최종 통합]
                        │
                  최종 산출물


## 에러 핸들링

| 상황 | 전략 |
|------|------|
| Task 1개 실패 | 1회 재시도. 재실패 시 해당 결과 없이 진행, 보고서에 누락 명시 |
| 타임아웃 | 현재까지 수집된 부분 결과 사용 |
| 데이터 충돌 | 출처 명시 후 병기, 삭제하지 않음 |

## 테스트 시나리오

### 정상 흐름
1. 사용자가 {입력}을 제공
2. Phase 1에서 {분석 결과} 도출
3. Phase 2에서 {N}개 Task 순차 실행
4. Phase 3에서 최종 결과 생성
5. 예상 결과: {output-path}/{filename} 생성

### 에러 흐름
1. Phase 2-2에서 {agent-2}가 실패
2. 1회 재시도 후에도 실패
3. {agent-2} 건너뛰고 Phase 2-3으로 진행
4. 최종 보고서에 "{agent-2} 미실행" 명시


---

## 작성 원칙

1. **실행 모드를 먼저 명시** — 오케스트레이터 상단에 "Task 병렬 호출" 또는 "Task 순차 호출" 명시
2. **Task 파라미터를 구체적으로** — name, prompt, model, run_in_background
3. **파일 경로는 절대적으로** — 상대 경로 금지, _workspace/ 기준 명확한 경로
4. **Phase 간 의존성 명시** — 어떤 Phase가 어떤 Phase의 결과에 의존하는지
5. **에러 핸들링은 현실적으로** — "모든 것이 성공한다"고 가정하지 않음
6. **테스트 시나리오 필수** — 정상 1 + 에러 1 이상

## 실제 오케스트레이터 참고

팬아웃/팬인 패턴의 오케스트레이터 기본 구조:
준비 → N개 Task 병렬 호출 → 결과 취합 → 통합 → 정리.
references/team-examples.md의 리서치 팀 예시를 참조.
