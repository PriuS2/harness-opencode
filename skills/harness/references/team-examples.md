# Task & Agent Examples

---

## 예시 1: 리서치 팀 (Task 병렬 호출)

### 아키텍처: 팬아웃/팬인
### 실행 모드: Task 병렬 호출


[parent/오케스트레이터]
    ├── Task({ name: "official", run_in_background: true })
    ├── Task({ name: "media", run_in_background: true })
    ├── Task({ name: "community", run_in_background: true })
    ├── Task({ name: "background", run_in_background: true })
    ├── 모든 Task 완료 대기
    └── 결과 취합 및 종합 보고서 생성


### 에이전트 구성

| 에이전트 | 역할 | 출력 |
|---------|------|------|
| official | 공식 문서/블로그 | _workspace/01_official.md |
| media | 미디어/투자 | _workspace/02_media.md |
| community | 커뮤니티/SNS | _workspace/03_community.md |
| background | 배경/경쟁/학술 | _workspace/04_background.md |
| (parent = 오케스트레이터) | 통합 보고서 | 종합보고서.md |

> 리서치 에이전트는 general-purpose 빌트인 타입을 사용하되, 반드시 .opencode/agents/{name}.md 파일로 정의한다.

### 오케스트레이터 워크플로우


Phase 1: 준비
  - 사용자 입력 분석 (주제, 조사 모드 파악)
  - _workspace/ 생성

> **모델 설정:** `model` 파라미터는 `provider/model` 형식으로 지정합니다. 사용자의 OpenCode 설정(config)에서 구성된 모델을 사용합니다. 예: `anthropic/claude-sonnet-4-20250514`, `openrouter/minimax/minimax-m2.7` 등

Phase 2: 병렬 조사 실행
  Task({ name: "official", prompt: "공식 채널 조사...", model: "{provider/model}", run_in_background: true })
  Task({ name: "media", prompt: "미디어/투자 동향 조사...", model: "{provider/model}", run_in_background: true })
  Task({ name: "community", prompt: "커뮤니티 반응 조사...", model: "{provider/model}", run_in_background: true })
  Task({ name: "background", prompt: "배경/경쟁 환경 조사...", model: "{provider/model}", run_in_background: true })

Phase 3: 결과 취합
  - 4개 산출물 Read
  - 종합 보고서 생성
  - 상충 정보는 출처 병기

Phase 4: 정리
  - _workspace/ 보존 (사후 검증·감사 추적용)


---

## 예시 2: SF 소설 집필 팀 (Task 병렬 + 순차)

### 아키텍처: 파이프라인 + 팬아웃
### 실행 모드: Task 병렬 → 순차 → 병렬 → 순차


Phase 1 (병렬): worldbuilder + character-designer + plot-architect
Phase 2 (순차): prose-stylist (집필)
Phase 3 (병렬): science-consultant + continuity-manager (리뷰)
Phase 4 (순차): prose-stylist (최종 수정)


### 에이전트 구성

| 에이전트 | 역할 | 스킬 |
|---------|------|------|
| worldbuilder | 세계관 구축 | world-setting |
| character-designer | 캐릭터 설계 | character-profile |
| plot-architect | 플롯 구조 | outline |
| prose-stylist | 문체 편집 + 집필 | write-scene, review-chapter |
| science-consultant | 과학 검증 | science-check |
| continuity-manager | 일관성 검증 | consistency-check |

### 에이전트 파일 예시: worldbuilder.md

markdown
---
name: worldbuilder
description: "SF 소설의 세계관을 구축하는 전문가. 물리 법칙, 사회 구조, 기술 수준, 역사를 설계한다."
---

# Worldbuilder — SF 세계관 설계 전문가

당신은 SF 소설의 세계관 설계 전문가입니다.

## 핵심 역할
1. 세계의 물리 법칙과 기술 수준 정의
2. 사회 구조, 정치 체계, 경제 시스템 설계
3. 역사적 맥락과 현재 갈등 구조 수립
4. 장소별 환경과 분위기 묘사

## 작업 원칙
- 내적 일관성 최우선 — 설정 간 모순이 없어야 한다
- "만약 이 기술이 있다면?" 연쇄 질문으로 세계의 파급 효과를 추론
- 이야기에 봉사하는 세계관 — 플롯을 방해하는 과도한 설정은 지양

## 입력/출력 프로토콜
- 입력: 사용자의 세계관 컨셉, 장르 요구사항
- 출력: _workspace/01_worldbuilder_setting.md
- 형식: 마크다운. 섹션별 (물리/사회/기술/역사/장소)

## 에러 핸들링
- 컨셉이 모호하면 3가지 방향을 제안하고 선택 요청
- 과학적 오류 발견 시 대안을 함께 제시


### 워크플로우 상세


Phase 1: Task 병렬 호출
  Task({ name: "worldbuilder", run_in_background: true })
  Task({ name: "character-designer", run_in_background: true })
  Task({ name: "plot-architect", run_in_background: true })
  → _workspace/에 산출물 저장

Phase 2: Task 순차 호출
  Task({ name: "prose-stylist", prompt: "_workspace/의 3개 산출물을 Read하여 집필" })
  → _workspace/02_prose_draft.md

Phase 3: Task 병렬 호출
  Task({ name: "science-consultant", run_in_background: true })
  Task({ name: "continuity-manager", run_in_background: true })
  → draft 검토, _workspace/에 리뷰 저장

Phase 4: Task 순차 호출
  Task({ name: "prose-stylist", prompt: "리뷰 결과 반영하여 최종 수정" })
  → 최종 산출물


---

## 예시 3: 웹툰 제작 팀 (Task 순차 - 생성-검증)

### 아키텍처: 생성-검증
### 실행 모드: Task 순차 호출

> 생성-검증 패턴에서 에이전트가 2개뿐이고, 순차 결과 전달이 핵심이므로 Task 순차가 적합.


Phase 1: Task(webtoon-artist) → 패널 생성
Phase 2: Task(webtoon-reviewer) → 검수
Phase 3: Task(webtoon-artist) → 문제 패널 재생성 (최대 2회)


### 에이전트 구성

| 에이전트 | 역할 | 스킬 |
|---------|------|------|
| webtoon-artist | 패널 이미지 생성 | generate-webtoon |
| webtoon-reviewer | 품질 검수 | review-webtoon, fix-webtoon-panel |

### 에이전트 파일 예시: webtoon-reviewer.md

markdown
---
name: webtoon-reviewer
description: "웹툰 패널의 품질을 검수하는 전문가. 구도, 캐릭터 일관성, 텍스트 가독성, 연출을 평가한다."
---

# Webtoon Reviewer — 웹툰 품질 검수 전문가

당신은 웹툰 패널의 품질을 검수하는 전문가입니다.

## 핵심 역할
1. 각 패널의 구도와 시각적 완성도 평가
2. 캐릭터 외형의 패널 간 일관성 검증
3. 말풍선 텍스트의 가독성과 배치 평가
4. 전체 에피소드의 연출 흐름과 페이싱 검토

## 작업 원칙
- PASS/FIX/REDO 3단계로 명확히 판정
- FIX는 부분 수정으로 해결 가능한 경우, REDO는 전면 재생성 필요
- 주관적 취향이 아닌 객관적 기준(일관성, 가독성, 구도)으로 판단

## 입력/출력 프로토콜
- 입력: _workspace/panels/ 디렉토리의 패널 이미지들
- 출력: _workspace/review_report.md

## 에러 핸들링
- 이미지 로드 실패 시 해당 패널을 REDO로 판정
- 2회 재생성 후에도 REDO인 패널은 경고와 함께 PASS 처리


### 에러 핸들링


재시도 정책:
- REDO 판정 패널 → artist에게 재생성 요청 (구체적 수정 지시 포함)
- 최대 2회 루프 후 강제 PASS
- 전체 패널의 50% 이상이 REDO면 사용자에게 프롬프트 수정 제안


---

## 예시 4: 코드 리뷰 팀 (Task 병렬 호출)

### 아키텍처: 팬아웃/팬인
### 실행 모드: Task 병렬 호출

> 코드 리뷰는 서로 다른 관점의 리뷰어들이 병렬로 독립적으로 분석하여 더 빠른 검토가 가능.


[parent] → Task 병렬 호출
    ├── Task({ name: "security-reviewer", run_in_background: true })
    ├── Task({ name: "performance-reviewer", run_in_background: true })
    └── Task({ name: "test-reviewer", run_in_background: true })
    → 리뷰어들이 결과를 parent에 반환
    → parent가 결과 종합


### 팀 통신 패턴


security → performance: "이 SQL 쿼리 주입 가능, 성능 측면에서도 확인 필요"
performance → test: "N+1 쿼리 발견, 관련 테스트 있는지 확인 부탁"
test → security: "인증 모듈 테스트 없음, 보안 관점에서 우선순위 의견?"
→ 모든 결과는 parent에서 취합


핵심: 각 리뷰어가 독립적으로 분석하고, 결과는 parent가 취합하여 종합.

---

## 예시 5: 감독자 패턴 — 코드 마이그레이션 팀 (Task 순차)

### 아키텍처: 감독자
### 실행 모드: Task 순차 호출


[supervisor] → Task 순차 호출
    ├→ Task({ name: "migrator-1" }) (batch A)
    ├→ Task({ name: "migrator-2" }) (batch B)
    └→ Task({ name: "migrator-3" }) (batch C)
    → 각 마이그레이터가 배칭된 파일 처리
    → supervisor가 최종 통합


### 에이전트 구성

| 에이전트 | 역할 |
|---------|------|
| (parent = migration-supervisor) | 파일 분석, 배치 분배, 진행 관리 |
| migrator-1~3 | 할당된 파일 배치를 마이그레이션 |

### 감독자의 동적 분배 로직


1. 전체 대상 파일 목록 수집
2. 복잡도 추정 (파일 크기, import 수, 의존성)
3. 파일 배치를 3개 그룹으로 분할
4. Task 순차 호출로 각 마이그레이터에 배칭 할당
5. 각 마이그레이터 완료 후 결과 취합
6. 모든 작업 완료 → 통합 테스트 실행


---

## 산출물 패턴 요약

### 에이전트 정의 파일
위치: 프로젝트/.opencode/agents/{agent-name}.md
필수 섹션: 핵심 역할, 작업 원칙, 입력/출력 프로토콜, 에러 핸들링

### 스킬 파일 구조
위치: 프로젝트/.opencode/skills/{skill-name}/SKILL.md (프로젝트 레벨)
또는: ~/.config/opencode/skills/{skill-name}/SKILL.md (글로벌 레벨)

### 통합 스킬 (오케스트레이터)
팀 전체를 조율하는 상위 스킬. 시나리오별 에이전트 구성과 워크플로우를 정의.
템플릿: references/orchestrator-template.md 참조.
**실행 모드를 반드시 명시** — Task 병렬 또는 Task 순차.
