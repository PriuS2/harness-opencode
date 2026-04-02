---
name: harness
description: "하네스를 구성합니다. 전문 에이전트를 정의하며, 해당 에이전트가 사용할 스킬을 생성하는 메타 스킬. (1) '하네스 구성해줘', '하네스 구축해줘' 요청 시, (2) '하네스 설계', '하네스 엔지니어링' 요청 시, (3) 새로운 도메인/프로젝트에 대한 하네스 기반 자동화 체계를 구축할 때, (4) 하네스 구성을 재구성하거나 확장할 때 사용."
---

# Harness — Agent & Skill Architect

도메인/프로젝트에 맞는 하네스를 구성하고, 각 에이전트의 역할을 정의하며, 에이전트가 사용할 스킬을 생성하는 메타 스킬.

**핵심 원칙:**
1. 에이전트 정의(.opencode/agents/ 또는 ~/.config/opencode/agents/)와 스킬(.opencode/skills/ 또는 ~/.config/opencode/skills/)을 생성한다.
2. **Task tool을 사용한 병렬/순차 호출을 기본 실행 모드로 사용한다.**
3. Claude 호환 경로(.claude/, ~/.claude/)도 지원한다.

## 워크플로우

### Phase 1: 도메인 분석
1. 사용자 요청에서 도메인/프로젝트 파악
2. 핵심 작업 유형 식별 (생성, 검증, 편집, 분석 등)
3. 기존 에이전트/스킬 확인 (충돌/중복 방지)
4. 프로젝트 코드베이스 탐색 — 기술 스택, 데이터 모델, 주요 모듈 파악
5. **사용자 숙련도 감지** — 대화의 맥락 단서(사용 용어, 질문 수준)로 기술 수준을 파악하고, 이후 커뮤니케이션 톤을 조절한다. 코딩 경험이 적은 사용자에게는 "assertion", "JSON schema" 같은 용어를 설명 없이 쓰지 않는다.

### Phase 2: 태스크 아키텍처 설계

#### 2-1. 실행 모드 선택: Task 병렬 vs 순차

**기본값은 Task 병렬 호출**이다. 2개 이상의 에이전트가 병렬로 독립 작업을 수행할 때는 Task를 run_in_background: true로 동시 호출한다. 각 Task 결과는 parent session에서 수신하여 취합한다.

순차 호출은 이전 Task의 결과가 다음 Task의 입력으로 필요한 경우에 선택한다.

#### 2-2. 아키텍처 패턴 선택

1. 작업을 전문 영역으로 분해
2. Task 호출 구조 결정 (아키텍처 패턴은 references/agent-design-patterns.md 참조)
   - **파이프라인**: 순차 의존 작업
   - **팬아웃/팬인**: 병렬 독립 작업
   - **전문가 풀**: 상황별 선택 호출
   - **생성-검증**: 생성 후 품질 검수
   - **감독자**: 중앙 에이전트가 상태 관리 및 동적 분배

#### 2-3. 에이전트 분리 기준

전문성·병렬성·컨텍스트·재사용성 4축으로 판단한다. 상세 기준표는 references/agent-design-patterns.md의 "에이전트 분리 기준" 참조.

#### Task tool 기본 파라미터

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| name | string | 호출할 에이전트 이름 |
| prompt | string | 에이전트에 전달할 지시사항 |
| model | string | provider/model 형식 (예: openrouter/minimax/minimax-m2.7) |
| tools | object | { tool_name: boolean } 형태로 도구 활성화/비활성화 |
| run_in_background | boolean | true면 병렬 실행, false면 순차 대기 |
| steps | number | 최대 반복 횟수 (선택) |

#### 에이전트 호출 방식

**1. Task tool (명시적 호출)**

Task({
  name: "explore",
  prompt: "코드베이스를 탐색해줘",
  model: "openrouter/minimax/minimax-m2.7",
  tools: { edit: false, write: false }
})


**2. @mention (메시지 내 직접 호출)**

@explore 이 코드베이스를 탐색해줘

- subagent를 직접 메시지에서 호출
- 결과를 parent session에서 수신

### Phase 3: 에이전트 정의 생성

**모든 에이전트는 반드시 .opencode/agents/{name}.md 파일로 정의한다.** Task tool의 prompt에 역할을 직접 넣는 것은 금지한다. 이유:
- 에이전트 정의가 파일로 존재해야 다음 세션에서 재사용 가능
- Task 간 협업 프로토콜이 명시되어야 결과 취합 품질 보장
- 하네스의 핵심 가치는 에이전트(누가)와 스킬(어떻게)의 분리

빌트인 타입(general-purpose, Explore, Plan)을 사용하더라도 에이전트 정의 파일은 생성한다. Task tool의 name 파라미터로 지정하고, 에이전트 정의 파일에는 역할·원칙을 담는다.

**모델 설정:** 모든 에이전트는 model: "openrouter/minimax/minimax-m2.7"을 사용한다. Task tool 호출 시 반드시 model 파라미터를 명시한다.

각 에이전트를 .opencode/agents/{name}.md에 정의한다. 필수 섹션: 핵심 역할, 작업 원칙, 입력/출력 프로토콜, 에러 핸들링.

> 정의 템플릿과 실제 파일 전문은 references/agent-design-patterns.md의 "에이전트 정의 구조" + references/team-examples.md 참조.

**QA 에이전트 포함 시 필수 사항:**
- QA 에이전트는 general-purpose 타입을 사용하라 (Explore는 읽기 전용이므로 검증 스크립트 실행 불가)
- QA의 핵심은 "존재 확인"이 아니라 **"경계면 교차 비교"** — API 응답과 프론트 훅을 동시에 읽고 shape을 비교
- QA는 전체 완성 후 1회가 아니라, **각 모듈 완성 직후 점진적으로 실행** (incremental QA)
- 상세 가이드: references/qa-agent-guide.md 참조

### Phase 4: 스킬 생성

각 에이전트가 사용할 스킬을 .opencode/skills/{name}/SKILL.md에 생성한다. 상세 작성 가이드는 references/skill-writing-guide.md 참조.

#### 4-1. 스킬 구조


skill-name/
├── SKILL.md (필수)
│   ├── YAML frontmatter (name, description 필수)
│   └── Markdown 본문
└── Bundled Resources (선택)
    ├── scripts/    - 반복/결정적 작업용 실행 코드
    ├── references/ - 조건부 로딩하는 참조 문서
    └── assets/     - 출력에 사용되는 파일 (템플릿, 이미지 등)


#### 4-2. Description 작성 — 적극적 트리거 유도

description은 스킬의 유일한 트리거 메커니즘이다. Claude는 트리거를 보수적으로 판단하는 경향이 있으므로, description을 **적극적("pushy")**으로 작성한다.

**나쁜 예:** "PDF 문서를 처리하는 스킬"
**좋은 예:** "PDF 파일 읽기, 텍스트/테이블 추출, 병합, 분할, 회전, 워터마크, 암호화, OCR 등 모든 PDF 작업을 수행. .pdf 파일을 언급하거나 PDF 산출물을 요청하면 반드시 이 스킬을 사용할 것."

핵심: 스킬이 하는 일 + 구체적 트리거 상황을 모두 기술하고, 유사하지만 트리거하면 안 되는 경우와 구분되도록 작성.

#### 4-3. 본문 작성 원칙

| 원칙 | 설명 |
|------|------|
| **Why를 설명하라** | "ALWAYS/NEVER" 같은 강압적 지시 대신, 왜 그렇게 해야 하는지 이유를 전달한다. LLM은 이유를 이해하면 엣지 케이스에서도 올바르게 판단한다. |
| **Lean하게 유지** | 컨텍스트 윈도우는 공공재다. SKILL.md 본문은 500줄 이내를 목표로, 무게를 벌지 않는 내용은 삭제하거나 references/로 이동한다. |
| **일반화하라** | 특정 예시에만 맞는 좁은 규칙보다, 원리를 설명하여 다양한 입력에 대응할 수 있게 한다. 오버피팅 금지. |
| **반복 코드는 번들링** | 테스트 실행에서 에이전트들이 공통으로 작성하는 스크립트가 발견되면 scripts/에 미리 번들링한다. |
| **명령형으로 작성** | "~한다", "~하라" 형태의 명령형/지시형 어조를 사용한다. |

#### 4-4. Progressive Disclosure (단계적 정보 공개)

스킬은 3단계 로딩 시스템으로 컨텍스트를 관리한다:

| 단계 | 로딩 시점 | 크기 목표 |
|------|----------|----------|
| **Metadata** (name + description) | 항상 컨텍스트에 존재 | ~100단어 |
| **SKILL.md 본문** | 스킬 트리거 시 | <500줄 |
| **references/** | 필요할 때만 | 무제한 (스크립트는 로딩 없이 실행 가능) |

**크기 관리 규칙:**
- SKILL.md가 500줄에 근접하면 세부 내용을 references/로 분리하고, 본문에 "언제 이 파일을 읽으라"는 포인터를 남긴다
- 300줄 이상의 reference 파일에는 상단에 **목차(ToC)**를 포함한다
- 도메인/프레임워크별 변형이 있으면 references/ 하위에 도메인별로 분리하여, 관련 파일만 로드한다

#### 4-5. 스킬-에이전트 연결 원칙

- 에이전트 1개 ↔ 스킬 1~N개 (1:1 또는 1:다)
- 여러 에이전트가 공유하는 스킬도 가능
- 스킬은 "어떻게 하는가"를 담고, 에이전트는 "누가 하는가"를 담는다

> 상세 작성 패턴, 예시, 데이터 스키마 표준은 references/skill-writing-guide.md 참조.

### Phase 5: 통합 및 오케스트레이션

오케스트레이터는 스킬의 특수한 형태로, 개별 에이전트와 스킬을 하나의 워크플로우로 엮어 전체를 조율한다. 구체적 템플릿은 references/orchestrator-template.md 참조.

#### 5-0. 모드별 오케스트레이터 패턴

**Task 병렬 호출 (기본):**
여러 Task를 run_in_background: true로 동시 호출하여 병렬 실행한다. 각 Task 결과는 parent session에서 수신하여 취합한다.


[오케스트레이터]
    ├── Task({ name: "agent-1", run_in_background: true })
    ├── Task({ name: "agent-2", run_in_background: true })
    ├── 모든 Task 완료 대기
    ├── 결과 취합 및 통합
    └── 최종 산출물 생성


**Task 순차 호출:**
이전 Task의 결과를 다음 Task의 입력으로 전달하며 순차 실행한다.


[오케스트레이터]
    ├── Task({ name: "agent-1" }) → 결과: _workspace/01_artifact.md
    ├── Task({ name: "agent-2", 입력: 01 결과 }) → 결과: _workspace/02_artifact.md
    └── 결과 통합


#### 5-1. 데이터 전달 프로토콜

| 전략 | 방식 | 적합한 경우 |
|------|------|-----------|
| **파일 기반** | _workspace/에 파일을 쓰고 읽음 | 대용량 데이터, 구조화된 산출물, 감사 추적 |
| **Task 결과 전달** | Task 결과를 다음 Task 입력으로 | 순차 의존성 있는 작업 |

파일 기반 전달 시 규칙:
- 작업 디렉토리 하위에 _workspace/ 폴더를 만들어 중간 산출물 저장
- 파일명 컨벤션: {phase}_{agent}_{artifact}.{ext} (예: 01_analyst_requirements.md)
- 최종 산출물만 사용자 지정 경로에 출력, 중간 파일(_workspace/)은 보존 (사후 검증·감사 추적용)

#### 5-2. 에러 핸들링

오케스트레이터 내에 에러 처리 방안을 포함한다. 핵심 원칙: 1회 재시도 후 재실패 시 해당 결과 없이 진행(보고서에 누락 명시), 상충 데이터는 삭제하지 않고 출처 병기.

> 에러 유형별 전략표와 구현 상세는 references/orchestrator-template.md의 "에러 핸들링" 참조.

#### 5-3. 팀 크기 가이드라인

| 작업 규모 | 권장 태스크 수 | 비고 |
|----------|------------|------|
| 소규모 (5~10개 작업) | 2~3개 병렬 Task | 태스크당 3~5개 작업 |
| 중규모 (10~20개 작업) | 3~5개 병렬 Task | 태스크당 4~6개 작업 |
| 대규모 (20개+ 작업) | 5~7개 병렬 Task | 태스크당 4~5개 작업 |

> 태스크가 많을수록 조율 오버헤드가 커진다. 3개의 집중된 태스크가 5개의 산만한 태스크보다 낫다.

### Phase 6: 검증 및 테스트

생성된 하네스를 검증한다. 상세 테스트 방법론은 references/skill-testing-guide.md 참조.

#### 6-1. 구조 검증

- 모든 에이전트 파일이 올바른 위치에 있는지 확인
- 스킬의 frontmatter(name, description) 검증
- Task 호출 패턴 일관성 확인

#### 6-2. 실행 모드별 검증

- Task 병렬 모드: run_in_background 설정, 결과 취합 경로 확인
- Task 순차 모드: Task 간 입력/출력 연결 확인

#### 6-3. 스킬 실행 테스트

1. **테스트 프롬프트 작성** — 각 스킬에 대해 2~3개의 현실적인 테스트 프롬프트를 작성한다.

2. **With-skill vs Without-skill 비교 실행** — 가능하면 스킬 있는 실행과 없는 실행을 병렬로 수행하여 스킬의 부가가치를 확인한다. Task를 두 개씩 실행한다:
   - **With-skill**: 스킬을 읽고 작업 수행
   - **Without-skill (baseline)**: 같은 프롬프트를 스킬 없이 수행

3. **결과 평가** — 산출물의 품질을 정성적(사용자 리뷰) + 정량적(assertion 기반) 으로 평가한다.

4. **반복 개선 루프** — 테스트 결과에서 문제가 발견되면 일반화하여 스킬을 수정하고 재테스트한다.

5. **반복 패턴 번들링** — 테스트 실행에서 공통으로 작성하는 코드를 scripts/에 미리 번들링한다.

#### 6-4. 트리거 검증

각 스킬의 description이 올바르게 트리거되는지 검증한다:

1. **Should-trigger 쿼리** (8~10개) — 스킬을 트리거해야 하는 다양한 표현
2. **Should-NOT-trigger 쿼리** (8~10개) — 키워드가 유사하지만 다른 도구/스킬이 적합한 "near-miss" 쿼리

**near-miss 작성 핵심:** "피보나치 함수 작성" 같이 명백히 무관한 쿼리는 테스트 가치가 없다. **경계가 모호한 쿼리**가 좋은 테스트 케이스다.

#### 6-5. 드라이런 테스트

- 오케스트레이터 스킬의 Phase 순서가 논리적인지 검토
- 데이터 전달 경로에 빈 구간(dead link)이 없는지 확인
- 모든 Task의 입력이 이전 Phase의 출력과 매칭되는지 확인

#### 6-6. 테스트 시나리오 작성

- 오케스트레이터 스킬에 ## 테스트 시나리오 섹션 추가
- 정상 흐름 1개 + 에러 흐름 1개 이상 기술

## 산출물 체크리스트

생성 완료 후 확인:

- [ ] .opencode/agents/ — **에이전트 정의 파일 필수 생성** (빌트인 타입이라도 파일 생성 필수)
- [ ] .opencode/skills/ — 스킬 파일들 (SKILL.md + references/)
- [ ] 오케스트레이터 스킬 1개 (데이터 흐름 + 에러 핸들링 + 테스트 시나리오 포함)
- [ ] 실행 모드 명시 (Task 병렬 또는 순차)
- [ ] 모든 Task 호출에 model: "openrouter/minimax/minimax-m2.7" 파라미터 명시
- [ ] 기존 에이전트/스킬과 충돌 없음
- [ ] 스킬 description이 적극적("pushy")으로 작성됨
- [ ] SKILL.md 본문이 500줄 이내, 초과 시 references/ 분리
- [ ] 테스트 프롬프트 2~3개로 실행 검증 완료
- [ ] 트리거 검증 (should-trigger + should-NOT-trigger) 완료

## 참고

- 하네스 패턴: references/agent-design-patterns.md
- 기존 하네스 예시: references/team-examples.md
- 오케스트레이터 템플릿: references/orchestrator-template.md
- **스킬 작성 가이드**: references/skill-writing-guide.md
- **스킬 테스트 가이드**: references/skill-testing-guide.md
- **QA 에이전트 가이드**: references/qa-agent-guide.md
