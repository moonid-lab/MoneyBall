# MoneyBall Workflow Stage Guide

Date: 2026-05-10  
Basis: static source reading only

이 문서는 MoneyBall의 5-step 워크플로우를 **사용자 관점 + 프론트엔드 동작 + 백엔드 동작 + 결과물** 기준으로 쉽게 풀어쓴 설명이다.  
핵심 기준은 `frontend/src/views/MainView.vue`, `frontend/src/components/Step1~Step5*.vue`, `backend/app/api/*.py`, `backend/app/services/*.py` 구현이다.

---

## 전체 흐름 한눈에 보기

1. **Upload docs + simulation requirement**
2. **Generate ontology / project**
3. **Build Zep graph**
4. **Create / prepare / run simulation**
5. **Generate report + interactive chat / survey**

쉽게 비유하면:

- 1단계: 재료 모으기
- 2단계: 분류 기준과 설계도 만들기
- 3단계: 실제 그래프 구조 만들기
- 4단계: 시뮬레이션 세계와 에이전트 준비 후 실행하기
- 5단계: 결과 보고서 만들고, 에이전트와 대화하기

---

## 1. Upload docs + simulation requirement

### 사용자가 보는 일

- 홈 화면에서 PDF / MD / TXT 파일을 올린다.
- "무엇을 예측하고 싶은지" 자연어로 simulation requirement를 입력한다.
- 시작 버튼을 누르면 바로 Process 화면으로 이동한다.

### 프론트엔드에서 하는 일

- `Home.vue`가 업로드 파일과 simulation requirement를 즉시 API로 보내지 않고,
  `frontend/src/store/pendingUpload.js`에 **임시 저장**한다.
- 이후 `MainView.vue`가 `projectId === 'new'` 상태에서 실제 업로드를 시작한다.

### 백엔드에서 하는 일

- 아직 이 단계 단독으로는 서버에 영구 저장되지 않는다.
- 실제 서버 처리는 다음 단계인 **Generate ontology / project** 호출 시 시작된다.

### 이 단계의 결과

- 브라우저 메모리 안에 “업로드 예정 데이터”가 잡혀 있는 상태
- 아직 `project_id`도 없고, 그래프도 없다

### 핵심 포인트

- 홈 화면은 “입력 수집” 역할이고,
- 실제 프로젝트 생성은 2단계 API에서 이뤄진다.

---

## 2. Generate ontology / project

### 한 줄 설명

문서를 읽고, **이번 사건을 어떤 종류의 인물/조직/관계로 해석할지 정의하는 단계**다.

### 사용자가 보는 일

- Process 화면에 들어오면 문서 업로드와 분석이 진행된다.
- 엔티티 타입, 관계 타입 같은 ontology 결과가 보이기 시작한다.

### 프론트엔드에서 하는 일

- `MainView.vue`가 `POST /api/graph/ontology/generate`를 호출한다.
- 요청은 `multipart/form-data`이며, 핵심 입력은:
  - `files[]`
  - `simulation_requirement`
  - 선택적으로 `project_name`, `additional_context`

### 백엔드에서 하는 일

`backend/app/api/graph.py`의 `generate_ontology()`가 아래 순서로 처리한다.

1. **프로젝트 생성**
   - `ProjectManager.create_project()`로 `proj_xxx` 형식의 `project_id`를 만든다.
   - 프로젝트 디렉터리와 파일 저장 경로를 만든다.

2. **파일 저장**
   - 업로드 파일을 프로젝트 폴더에 저장한다.
   - 파일명은 안전한 랜덤 파일명으로 바뀌고, 원래 파일명은 메타데이터로 유지된다.

3. **텍스트 추출 및 전처리**
   - PDF / MD / TXT에서 텍스트를 뽑는다.
   - 여러 파일이면 하나의 큰 텍스트로 합친다.
   - 합쳐진 텍스트는 프로젝트용 `extracted_text.txt`로 저장된다.

4. **LLM으로 ontology 생성**
   - `OntologyGenerator.generate()`가 문서 내용과 simulation requirement를 LLM에 전달한다.
   - 여기서 생성되는 것은 실제 인물이 아니라 **타입 정의**다.
   - 예:
     - 엔티티 타입: `Student`, `Professor`, `University`, `MediaOutlet`
     - 관계 타입: `STUDIES_AT`, `REPORTS_ON`, `RESPONDS_TO`

5. **결과 검증 및 정리**
   - 엔티티 이름은 PascalCase로 강제 정리된다.
   - 관계 이름은 UPPER_SNAKE_CASE로 정리된다.
   - 중복 타입 제거, fallback 타입 보정 같은 후처리가 들어간다.

6. **project 상태 저장**
   - `project.ontology`
   - `project.analysis_summary`
   - `project.status = ontology_generated`

### 이 단계의 결과물

- `project_id`
- 업로드 파일 목록
- 추출된 전체 텍스트
- ontology (entity types / edge types)
- analysis summary

### 쉽게 이해하기

이 단계는 **“문서 속 내용을 어떤 규칙으로 읽을지 정하는 단계”**다.  
아직 Zep graph는 만들어지지 않았고, 먼저 **설계도와 분류표를 만드는 것**에 가깝다.

### 왜 이 단계가 필요한가

문서를 바로 그래프로 넣으면 시스템이:

- 누가 사람인지
- 누가 조직인지
- 어떤 관계를 중요하게 봐야 하는지

를 일관되게 판단하기 어렵다.  
그래서 먼저 **“이번 문서를 읽는 기준”**을 만들고, 그 기준으로 다음 단계에서 그래프를 만든다.

### 다음 단계로 넘기는 핵심 데이터

- `project_id`
- `project.ontology`
- `extracted_text`

---

## 3. Build Zep graph

### 한 줄 설명

2단계에서 만든 ontology를 기준으로, **전체 문서를 실제 Zep 지식 그래프로 변환하는 단계**다.

### 사용자가 보는 일

- 그래프 빌드 진행률이 올라간다.
- 노드/관계 수가 생기고, 그래프 시각화가 나타난다.

### 프론트엔드에서 하는 일

- `MainView.vue`가 `POST /api/graph/build`를 호출한다.
- 이후 `GET /api/graph/task/:taskId`로 진행률을 폴링한다.
- 완료되면 `GET /api/graph/data/:graphId`로 실제 그래프 데이터를 불러온다.

### 백엔드에서 하는 일

`backend/app/api/graph.py`의 `build_graph()`가 아래 순서로 처리한다.

1. **project 조회**
   - `project_id`로 프로젝트를 가져온다.
   - ontology가 생성되어 있는지, 추출된 텍스트가 있는지 확인한다.

2. **빌드 작업 시작**
   - `TaskManager`에 비동기 task를 만든다.
   - `project.status = graph_building`
   - `project.graph_build_task_id = task_id`

3. **Zep graph 생성**
   - `GraphBuilderService.create_graph()`가 `mirofish_xxx` 형식의 `graph_id`를 만든다.
   - 이 시점부터 “실제 그래프 저장소”가 생긴다.

4. **ontology를 Zep에 등록**
   - `GraphBuilderService.set_ontology()`가 엔티티 타입/관계 타입을 Zep schema로 넣는다.
   - 즉, 그래프가 “어떤 타입 체계로 동작할지”를 Zep에 알려준다.

5. **텍스트를 chunk로 분할**
   - 긴 문서를 바로 넣지 않고 잘게 나눈다.
   - 기본값은 대략:
     - `chunk_size = 500`
     - `chunk_overlap = 50`

6. **chunk들을 배치로 Zep에 전송**
   - `add_text_batches()`가 문서 조각들을 Zep에 넣는다.
   - Zep은 이 조각들을 읽으며:
     - 엔티티를 찾고
     - 관계를 찾고
     - 노드와 엣지를 생성한다

7. **Zep 처리 완료까지 대기**
   - 각 batch(episode)가 처리 완료되었는지 기다린다.
   - 프론트는 task progress를 계속 폴링한다.

8. **그래프 데이터 조회 및 상태 완료 처리**
   - `builder.get_graph_data(graph_id)`로 노드/엣지 정보를 가져온다.
   - `project.status = graph_completed`
   - 최종 결과에 `graph_id`, `node_count`, `edge_count`, `chunk_count`가 들어간다.

### 이 단계의 결과물

- `graph_id`
- Zep에 저장된 실제 graph
- 노드 / 관계 수
- 프론트에서 시각화 가능한 graph data

### 쉽게 이해하기

이 단계는 **설계도를 보고 실제 구조물을 짓는 단계**다.

- 2단계 ontology = 설계도
- 3단계 Zep graph = 실제 건물

즉, 2단계가 “어떤 타입이 있는지 정의”했다면,  
3단계는 “문서 전체를 읽어 실제 인물/조직/관계를 그래프 노드/엣지로 만든다.”

### 다음 단계로 넘기는 핵심 데이터

- `project_id`
- `graph_id`
- graph data

---

## 4. Create / prepare / run simulation

### 한 줄 설명

완성된 graph를 바탕으로 **시뮬레이션 인스턴스를 만들고, 에이전트 프로필과 행동 규칙을 생성한 뒤, 실제로 실행하는 단계**다.

### 크게 세 부분으로 나뉜다

1. **create simulation**
2. **prepare simulation**
3. **start simulation**

---

### 4-1. Create simulation

#### 프론트엔드

- `Step1GraphBuild.vue`가 `POST /api/simulation/create`를 호출한다.

#### 백엔드

- `simulation_id = sim_xxx`를 만든다.
- `project_id`, `graph_id`, 플랫폼 사용 여부를 묶어 시뮬레이션 상태를 만든다.

#### 결과

- 아직 실행은 안 됨
- 단지 “이 graph를 기반으로 돌릴 시뮬레이션 인스턴스”가 생긴 상태

---

### 4-2. Prepare simulation

#### 프론트엔드

- `Step2EnvSetup.vue`가 `POST /api/simulation/prepare`를 호출한다.
- 이후 `prepare/status`를 폴링한다.
- 동시에 프로필과 config를 실시간으로 읽어 화면에 보여준다.

#### 백엔드

`backend/app/api/simulation.py`와 `SimulationManager`가 아래 일을 한다.

1. **이미 준비된 시뮬레이션인지 확인**
   - `state.json`, profile 파일, config 파일 등을 보고 재생성이 필요한지 판단한다.

2. **Zep graph에서 엔티티 읽기**
   - 그래프에서 simulation에 쓸 실제 개체들을 읽어온다.
   - 필요하면 특정 entity type만 골라서 사용할 수 있다.

3. **에이전트 프로필 생성**
   - 각 엔티티를 OASIS Agent Profile로 변환한다.
   - 예: 이름, 직업, 관심사, 성향, bio 등

4. **시뮬레이션 설정 생성**
   - LLM이 전체 simulation config를 만든다.
   - 예:
     - 총 시뮬레이션 시간
     - round당 시간
     - 시간대별 활동량
     - agent별 posting 빈도
     - sentiment bias
     - influence weight

5. **파일 저장**
   - `backend/uploads/simulations/<simulation_id>/` 아래에
     - `state.json`
     - `reddit_profiles.json`
     - `twitter_profiles.csv`
     - `simulation_config.json`
     등을 저장한다.

#### 결과

- “시뮬레이션을 돌릴 준비”가 끝난 상태
- 각 agent의 persona와 행동 규칙이 준비됨

#### OASIS가 무엇인가

- OASIS는 이 프로젝트가 사용하는 **소셜 상호작용 시뮬레이션 엔진**이다.
- Zep graph가 “누가 누구와 어떤 관계인지”를 담고 있다면,
  OASIS는 그 관계를 가진 agent들이 실제로 **포스트하고, 댓글 달고, 반응하는 실행 환경**이다.
- README 기준 명칭은 **Open Agent Social Interaction Simulations**다.

#### OASIS Agent Profile이 무엇인가

- OASIS Agent Profile은 “가상 SNS 페르소나 계정”을 OASIS가 읽을 수 있는 **프로필 데이터 구조**로 만든 것이다.
- 일반적인 업계 용어라기보다, 이 프로젝트와 OASIS 실행 맥락에서 쓰는 **구현 명칭**에 가깝다.
- Zep graph의 엔티티를 그대로 시뮬레이션에 넣는 것이 아니라, 아래처럼 **행동 가능한 계정 정보**로 바꿔 넣는다.
  - 공통 필드: `user_id`, `username`, `name`, `bio`, `persona`
  - 추가 persona 정보: `profession`, `interested_topics`, `age`, `mbti`, `country`
  - 플랫폼용 필드:
    - Reddit형: `karma`
    - Twitter형: `friend_count`, `follower_count`, `statuses_count`

#### 왜 OASIS Agent Profile이 필요한가

- graph 엔티티만으로는 “누가 누구와 관계가 있는지”는 알 수 있어도,
  시뮬레이션 엔진이 바로 사용할 **계정 정체성, 말투, 관심사, 플랫폼용 메타데이터**는 부족하다.
- 그래서 prepare 단계에서 엔티티를:
  - 실제 행동 주체(agent)
  - SNS 계정 형식
  - 인터뷰/대화에 응답 가능한 persona
  로 변환한다.

쉽게 말하면:

- Zep entity = 그래프 속 개체
- OASIS Agent Profile = 시뮬레이션에서 행동할 캐릭터 시트

예:

- `Professor` 엔티티 -> 교수 개인 계정
- `University` 엔티티 -> 대학 공식 계정 또는 대표 계정
- `Organization` 엔티티 -> 단체 대표 계정

즉, 실제 외부 SNS 계정을 연결하는 것이 아니라,
**시뮬레이션 세계 안에서 발언하고 반응할 가상 계정**을 만든다고 이해하면 된다.

#### simulation config에서 사용자가 바꿀 수 있는 값

현재 구조는 **세부 config를 사용자가 직접 편집하는 방식이 아니라, 대부분 자동 생성하는 방식**이다.

- 사용자가 실질적으로 조절하는 핵심 입력:
  - `simulation_requirement`
  - 실행 시점의 `max_rounds`(선택 시)
- API 수준에서 받을 수 있지만 현재 UI에서는 사실상 고정에 가까운 값:
  - `enable_twitter`, `enable_reddit`
  - `use_llm_for_profiles`
  - `parallel_profile_count`
  - `entity_types`
- 자동 생성되는 값:
  - `total_simulation_hours`
  - `minutes_per_round`
  - `agents_per_hour_min/max`
  - `peak_hours`, `off_peak_hours`
  - agent별 `posts_per_hour`, `comments_per_hour`
  - `active_hours`
  - `sentiment_bias`
  - `stance`
  - `influence_weight`
  - `initial_posts`, `scheduled_events`, `hot_topics`
  - platform별 추천/확산 관련 설정

즉, 현재는 **사용자가 목표와 대략적인 실행 범위만 주고, 상세 simulation config는 시스템이 생성**하는 구조다.

---

### 4-3. Start simulation

#### 프론트엔드

- `Step3Simulation.vue`가 `POST /api/simulation/start`를 호출한다.
- 이후:
  - `GET /api/simulation/:id/run-status`
  - `GET /api/simulation/:id/run-status/detail`
  를 주기적으로 폴링한다.

#### 백엔드

- `SimulationRunner`가 백그라운드에서 실행된다.
- Twitter형/Reddit형 두 플랫폼을 병렬로 돌린다.
- 각 round에서 agent들이 어떤 행동을 했는지 수집한다.
  - `CREATE_POST`
  - `COMMENT`
  - `LIKE`
  - `REPOST`
  - `FOLLOW`
  - `SEARCH`
  - `DO_NOTHING`

#### 결과

- 시뮬레이션 타임라인
- agent action 로그
- 플랫폼별 진행 상태
- 이후 report 생성에 사용할 행동 데이터

#### round의 의미

- round는 시뮬레이션의 **기본 진행 단위**, 즉 한 번의 턴(turn)이다.
- 현실 시간 1초를 의미하는 것이 아니라,
  **시뮬레이션 세계에서 시간이 한 칸 진행되는 단위**라고 보면 된다.
- 각 round마다:
  - 현재 시점에 활성화될 agent가 선택되고
  - agent들이 post / comment / like / repost / follow / search / idle 같은 행동을 하고
  - 그 결과가 다음 round의 입력으로 이어진다.

예를 들어 `minutes_per_round = 30`이면:

- 1 round = 시뮬레이션 시간 30분
- 10 rounds = 시뮬레이션 시간 5시간
- 40 rounds = 시뮬레이션 시간 20시간

따라서 `max_rounds`는 “이 시뮬레이션을 몇 턴까지 돌릴 것인가”를 의미한다.

### 쉽게 이해하기

이 단계는 **등장인물을 캐스팅하고, 세계관 설정을 세운 뒤, 실제로 시뮬레이션을 돌리는 단계**다.

---

## 5. Generate report + interactive chat / survey

### 한 줄 설명

시뮬레이션 결과를 바탕으로 **보고서를 생성하고**, 이후 **Report Agent 또는 개별 Agent와 대화하는 단계**다.

### 크게 두 부분으로 나뉜다

1. **generate report**
2. **interactive chat / survey**

---

### 5-1. Generate report

#### 프론트엔드

- `Step3Simulation.vue`가 완료 후 `POST /api/report/generate`를 호출한다.
- `report_id`를 받으면 `/report/:reportId` 화면으로 이동한다.
- `Step4Report.vue`는:
  - `GET /api/report/:reportId/agent-log`
  - `GET /api/report/:reportId/console-log`
  를 폴링하며 진행 상황을 보여준다.

#### 백엔드

`backend/app/api/report.py`와 `ReportAgent`가 아래 일을 한다.

1. **report task 시작**
   - `report_id = report_xxx` 생성
   - 비동기 task 생성

2. **Report Agent 생성**
   - 입력:
     - `graph_id`
     - `simulation_id`
     - `simulation_requirement`

3. **보고서 outline 계획**
   - 어떤 섹션으로 보고서를 쓸지 먼저 설계한다.

4. **섹션별 생성**
   - 각 섹션마다 ReACT 스타일로 사고/도구 호출/초안 작성 과정을 거친다.
   - 사용 가능한 도구 예:
     - `insight_forge`
     - `panorama_search`
     - `quick_search`
     - `interview_agents`

5. **결과 저장**
   - `backend/uploads/reports/<report_id>/` 아래에
     - 메타데이터
     - 진행 상태
     - 섹션별 markdown
     - 전체 보고서
     - agent 로그
     - console 로그
     가 저장된다.

#### 결과

- outline이 있는 보고서
- section별 markdown
- report agent 실행 로그

### 쉽게 이해하기

이 단계는 **“시뮬레이션에서 무슨 일이 일어났는지 분석 보고서로 정리하는 단계”**다.

---

### 5-2. Interactive chat / survey

#### 프론트엔드

`Step5Interaction.vue`에서 두 가지 모드가 있다.

1. **Report Agent와 채팅**
   - `POST /api/report/chat`
   - 사용자가 질문하면 Report Agent가 보고서/그래프/시뮬레이션 문맥을 바탕으로 답한다.

2. **개별 Agent 인터뷰 / 설문**
   - `POST /api/simulation/interview/batch`
   - 특정 agent 하나와 대화할 수도 있고,
   - 여러 agent를 골라 같은 질문을 던지는 survey도 가능하다.

#### 백엔드

- Report Agent chat:
  - 질문을 받아 필요한 검색 도구를 호출하고 응답한다.
- Agent interview:
  - 특정 simulation agent에게 질문을 전달하고
  - 그 agent의 persona / memory / action context를 바탕으로 답하게 한다.

#### 결과

- 보고서 기반 후속 질의응답
- 개별 에이전트 관점의 답변
- 여러 agent의 집단 반응 비교

### 쉽게 이해하기

이 단계는 **정적인 보고서를 읽는 것에서 끝나지 않고**,  
“왜 이런 결과가 나왔는지”를 다시 캐묻는 탐색 단계다.

---

## 단계 간 데이터 의존성

각 단계는 앞 단계 결과를 그대로 이어받는다.

| 단계 | 다음 단계로 넘기는 핵심 값 |
|---|---|
| 1 -> 2 | 업로드 파일, simulation requirement |
| 2 -> 3 | `project_id`, `ontology`, `extracted_text` |
| 3 -> 4 | `graph_id`, graph data |
| 4 -> 5 | `simulation_id`, action logs, prepared profiles/config |
| 5 -> interaction | `report_id`, `simulation_id`, report artifacts |

즉:

- ontology 없이 graph build 불가
- graph 없이 simulation 생성 불가
- simulation 없이 report 생성 불가
- report 완료 후에 interaction 기능이 더 풍부해짐

---

## 가장 중요한 이해 포인트

### 1. `project`와 `simulation`은 다르다

- `project`는 문서와 ontology, graph build의 중심
- `simulation`은 graph를 기반으로 실제 실행되는 시뮬레이션 인스턴스

### 2. ontology와 graph는 다르다

- ontology = 타입 정의
- graph = 실제 엔티티/관계 데이터

### 3. simulation prepare와 simulation run은 다르다

- prepare = agent persona + config 생성
- run = 실제 액션 발생

### 3-1. OASIS Agent Profile은 일반명사라기보다 실행용 포맷이다

- “가상 SNS 페르소나 계정”은 개념 설명이고,
- `OASIS Agent Profile`은 그 개념을 OASIS 엔진이 읽을 수 있게 만든 **실행용 프로필 구조**다.

### 4. report는 단순 요약이 아니다

- Report Agent가 tool을 써서 그래프/시뮬레이션 결과를 다시 탐색하면서 작성한다.

---

## 요약

MoneyBall의 5-step 워크플로우는 다음처럼 이해하면 가장 쉽다.

1. **문서를 넣고 질문을 정한다**
2. **문서를 읽는 기준(ontology / project)을 만든다**
3. **그 기준으로 Zep graph를 만든다**
4. **graph 기반으로 agent 세계를 만들고 시뮬레이션을 돌린다**
5. **결과를 보고서로 만들고, agent들과 추가로 대화한다**

이 구조를 이해하면 프론트/API/백엔드 서비스가 왜 이렇게 나뉘어 있는지 자연스럽게 보인다.
