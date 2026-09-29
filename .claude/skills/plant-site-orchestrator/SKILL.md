---
name: plant-site-orchestrator
description: "식물 도감 홈페이지 제작을 위한 콘텐츠 큐레이터/사이트 빌더/QA 파이프라인을 조율. '식물 도감 홈페이지 만들어줘', '식물 사이트 구축해줘' 같은 초기 요청뿐 아니라 '식물 정보 추가/수정', '이번 달 추천 식물 바꿔줘', '사이트 업데이트/리디자인', '다시 만들어줘', '검증해줘' 등 후속 요청에도 반드시 이 스킬을 사용."
---

# Plant Site Orchestrator

식물 도감 홈페이지(분류·물주기·개화 시기·관리 팁·이번 달 추천 식물)를 만들고 유지보수하는 파이프라인을 조율한다.

## 실행 모드: 서브 에이전트 (파이프라인)

이 환경에는 `TeamCreate`/`TaskCreate` 같은 에이전트 팀 전용 도구가 없고, `.claude/agents/*.md` 커스텀 에이전트도 `subagent_type`으로 직접 호출할 수 없다(사용 가능한 타입: `claude, claude-code-guide, Explore, general-purpose, Plan, statusline-setup`). 대신 `Agent` 도구를 `subagent_type: "general-purpose"`로 호출하면서, 프롬프트 맨 앞에 "`.claude/agents/{name}.md`를 Read로 먼저 읽고 그 역할·원칙을 따르라"는 지시를 넣어 페르소나를 주입한다. 산출물은 파일(`_workspace/`)로 다음 단계에 전달한다. 콘텐츠가 있어야 화면을 만들 수 있고, 화면이 있어야 검증할 수 있는 순차 의존 구조라 파이프라인이 자연스럽다.

## 에이전트 구성

| 순서 | 페르소나 파일 | subagent_type | 역할 | 스킬 | 출력 |
|------|---------------|----------------|------|------|------|
| 1 | `.claude/agents/plant-content-curator.md` | `general-purpose` | 식물 데이터 작성 | `plant-content` | `_workspace/01_content/plants.json` |
| 2 | `.claude/agents/plant-site-builder.md` | `general-purpose` | 홈페이지 Artifact 제작 | `plant-site-builder` (+ `frontend-design`, `artifact-design`) | `_workspace/02_site/artifact_url.txt` |
| 3 | `.claude/agents/plant-qa-inspector.md` | `general-purpose` | 데이터-화면 정합성 검증 | `plant-qa` | `_workspace/03_qa/report.md` |

모든 `Agent` 호출에 `model: "opus"`를 명시하고, 프롬프트 시작에 해당 페르소나 파일 경로를 Read로 읽으라는 지시를 포함한다. (이 환경에서 커스텀 `subagent_type`이 지원되기 시작하면 페르소나 파일명과 동일한 `subagent_type`을 직접 지정하는 방식으로 전환한다.)

## 워크플로우

### Phase 0: 컨텍스트 확인
1. `_workspace/` 존재 여부를 확인한다.
2. 판단:
   - **미존재** → 초기 실행. Phase 1로 진행
   - **존재 + 부분 수정 요청**(식물 1종 추가, 이번 달 추천 교체, 특정 섹션 수정 등) → 해당 에이전트만 재호출. 이전 산출물 경로를 프롬프트에 포함해 "기존 결과를 읽고 그 부분만 반영"하도록 지시
   - **존재 + 완전히 새로운 컨셉 요청** → 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동한 뒤 새 `_workspace/` 생성

### Phase 1: 콘텐츠 생성
`Agent(subagent_type: "plant-content-curator", model: "opus", prompt: "<요청 내용 + 오늘 날짜>")`를 호출해 `_workspace/01_content/plants.json`을 생성/갱신한다.

### Phase 2: 사이트 제작
Phase 1 완료 후 `Agent(subagent_type: "plant-site-builder", model: "opus", prompt: "_workspace/01_content/plants.json을 읽고 홈페이지를 Artifact로 퍼블리시하라. 기존 URL이 있으면 _workspace/02_site/artifact_url.txt에서 확인해 같은 URL에 업데이트하라.")`를 호출한다.

### Phase 3: 정합성 검증
Phase 2 완료 후 `Agent(subagent_type: "plant-qa-inspector", model: "opus", prompt: "...")`를 호출해 `_workspace/03_qa/report.md`를 생성한다.

### Phase 4: 수정 루프
1. `report.md`에 실패 항목이 있으면, 원인에 따라 Phase 1(`plant-content-curator`) 또는 Phase 2(`plant-site-builder`)를 재호출한다 — 이때 report.md의 구체적 지적 내용을 프롬프트에 포함한다.
2. 수정 후 Phase 3(QA)을 다시 실행한다.
3. **최대 2회**까지 재시도한다. 2회 후에도 실패 항목이 남으면 수정을 멈추고 사용자에게 남은 문제를 보고한다 — 무한 루프 방지.

### Phase 5: 결과 보고
1. Artifact URL을 사용자에게 전달한다.
2. QA 통과/실패 현황을 요약한다(남은 실패가 있으면 명시).
3. `_workspace/`는 삭제하지 않는다 — 다음 요청에서 재사용/부분 수정에 필요하다.

## 데이터 흐름

```
[콘텐츠 큐레이터] → plants.json → [사이트 빌더] → Artifact URL → [QA 검증] → report.md
        ↑                                ↑                              │
        └──────────── 실패 시 원인에 따라 재호출 (최대 2회) ──────────────┘
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 콘텐츠 큐레이터가 특정 식물 정보를 확신 못해 제외 | 정상 진행, 최종 보고에 제외 목록 포함 |
| 사이트 빌더가 Artifact 퍼블리시 실패 | 1회 재시도. 재실패 시 사용자에게 알리고 로컬 HTML 산출물만 보고 |
| QA가 Artifact URL을 읽지 못함 | 검증 중단, 사용자에게 즉시 보고 (화면 없이 검증한 것처럼 꾸미지 않음) |
| 2회 수정 루프 후에도 실패 항목 존재 | 남은 문제를 사용자에게 명시하고 진행 여부를 묻는다 |

## 테스트 시나리오

### 정상 흐름
1. 사용자가 "식물 도감 홈페이지 만들어줘, 물주기/분류/개화시기/관리팁/이번 달 추천 포함"을 요청
2. Phase 0에서 `_workspace/` 없음 확인 → 초기 실행
3. Phase 1에서 콘텐츠 큐레이터가 10종 이상의 `plants.json` 생성
4. Phase 2에서 사이트 빌더가 Artifact 퍼블리시, URL 저장
5. Phase 3에서 QA가 5개 경계면을 검증해 report.md 생성
6. 실패 항목 없으면 Phase 5로 바로 진행, 사용자에게 URL 전달

### 에러 흐름
1. Phase 3에서 QA가 "카테고리 필터에 '다육식물'이 데이터에 있으나 화면 필터에 없음" 발견
2. Phase 4에서 `plant-site-builder`를 해당 지적과 함께 재호출
3. 사이트 빌더가 필터 로직 수정 후 재퍼블리시
4. QA 재검증 → 통과 → Phase 5로 진행

## 후속 작업 예시
- "몬스테라 추가해줘" → Phase 0에서 부분 수정 감지 → 콘텐츠 큐레이터만 재호출 → 사이트 빌더가 갱신된 데이터 반영 → QA 재검증
- "이번 달 추천 식물 바꿔줘" → 콘텐츠 큐레이터가 `monthly_pick`만 수정 → 이후 동일 파이프라인
- "전체적으로 톤 다시 잡아줘" → 사이트 빌더만 재호출(데이터는 그대로)
