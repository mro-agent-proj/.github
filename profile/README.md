<div align="center">

<img src="assets/logo.png" width="120" alt="MRO Agent 로고">

# MRO Agent

### 항공기 정비요청 AI Agent

<sub>정비 신고를 읽고, 근거를 찾고, 맞는 정비팀에 배정합니다</sub>

**"근거 없이는 배정하지 않는다."**

현장에서 올라온 정비 신고, 어느 정비팀에 보내야 할까요?<br>
MRO Agent는 같은 기체의 수리이력과 **FAA 정비 결함 사례 71,741건**을 근거로 담당 부서를 추천하고,<br>
근거가 검증될 때만 자동으로 배정합니다. 확신이 없으면 사람에게 넘깁니다.

[![통합 레포](https://img.shields.io/badge/시작하기-integration-181717?style=for-the-badge&logo=github)](https://github.com/mro-agent-proj/integration)
[![아키텍처](https://img.shields.io/badge/AWS-아키텍처-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://github.com/mro-agent-proj/deliverables/tree/main/design#aws-배포-구성)
[![보고서](https://img.shields.io/badge/최종-결과보고서-4B5563?style=for-the-badge)](https://github.com/mro-agent-proj/deliverables/tree/main/report)
[![설계서](https://img.shields.io/badge/아키텍처-설계서-2563EB?style=for-the-badge)](https://github.com/mro-agent-proj/deliverables/tree/main/design)

</div>

<br>

## 30초 요약

> **문제** — 정비 신고가 들어오면 사람이 증상을 읽고, 비슷한 과거 사례를 찾고, 담당 정비팀을 정해야 합니다. 신고가 많을수록 **분류와 근거 찾기**에 시간이 걸리고, 담당자마다 판단이 달라집니다.
>
> **해결** — AI Agent가 신고를 받아 **같은 기체의 수리이력**을 먼저 비교하고, 부족하면 **SDR(정비 결함 보고) 사례와 기술자료**를 직접 검색해 담당 부서를 추천합니다.
>
> **결과** — 인용한 근거가 실제로 조회된 자료인지 백엔드가 다시 검증합니다. 통과하면 **자동 배정**, 아니면 **사람 검토**로 넘어갑니다.

<br>

<details>
<summary><strong>프로젝트 배경·문제 정의·MVP 자세히 보기</strong></summary>

## 왜 만들었나요?

정비가 처음이라면 이렇게 생각해 보세요.

> **응급실 접수**를 떠올려 보세요.<br>
> 환자가 "가슴이 답답해요"라고 말하면, 접수 간호사는 증상과 과거 진료 기록을 보고 어느 과로 보낼지 정합니다.<br>
> 잘못 보내면 다른 과로 다시 옮겨야 하고, 그만큼 치료가 늦어집니다.

항공기 정비도 같습니다. "엔진 시동 시 진동 발생" 같은 신고 한 줄을 보고 **어느 계통(ATA)의 문제인지, 어느 정비팀이 맡아야 하는지**를 정해야 합니다.

<table>
<tr>
<td align="center" width="33%"><h2>71,741건</h2>검색에 쓰는 FAA 정비 결함 보고(SDR)</td>
<td align="center" width="33%"><h2>1,600건</h2>기체 20대의 수리이력</td>
<td align="center" width="33%"><h2>8개</h2>배정 대상 정비 부서</td>
</tr>
</table>

- **같은 증상이라도 기체 이력에 따라 원인이 다릅니다.** 그래서 먼저 같은 기체의 과거 수리를 봅니다.
- **AI의 답은 틀릴 수 있습니다.** 그래서 AI가 인용한 근거를 코드로 다시 확인하고, 확인되지 않으면 배정하지 않습니다.
- **판단 과정이 남아야 합니다.** 정비팀은 AI가 왜 그렇게 판단했는지, 어떤 사례를 봤는지 화면에서 바로 봅니다.

<br>

## 문제와 AI 적용 지점

| 항목 | 내용 |
|:--|:--|
| **비즈니스 문제** | 정비 신고마다 사람이 증상을 읽고 과거 사례를 찾아 담당 부서를 정해야 해서, 분류에 시간이 들고 담당자마다 판단이 달라짐 |
| **대상 사용자** | 신고자(항공사 정비 요청자), 정비팀(부서 8개), 관리자 |
| **유저 문제** | 신고자는 누가 언제 처리하는지 모르고, 정비팀은 왜 우리 팀에 왔는지 근거를 다시 찾아야 하며, 관리자는 애매한 건만 골라 보기 어려움 |
| **AI 적용 지점** | 증상 → ATA 계통 분류(LLM1), 같은 기체 이력·SDR 사례·기술자료 검색(MCP + RAG), 판단 대조(LLM3). 근거가 없으면 사람 검토로 |
| **핵심 시나리오** | 신고 접수 → 기체·기종 검증 → AI 분류·조사·검토 → 근거 재검증 → 정비팀 배정 → 수락 |
| **데이터** | FAA 공개 SDR 71,741건, JASC 548개, AMM 참조 35,396건, Airbus Safety First 61편 + 시연용 업무 시드(기체 20, 수리이력 1,600). 계정은 시연용 `.test` 주소뿐이고 실제 개인정보 없음 |
| **성공 지표** | 서비스 평가 200건의 자동배정 정밀도·오배정률·사람 검토 비율, 시스템 오류·시간 초과 0건 |

<br>

## 이번 프로젝트의 MVP

핵심 가치 하나를 끝까지 동작하게 만드는 것이 목표였습니다.

| | 항목 | 이 프로젝트에서 |
|:--:|:--|:--|
| **1** | **가치를 한 문장으로** | 정비 신고를 **근거와 함께** 담당 정비팀에 바로 배정한다 |
| **2** | **그 가치가 끝까지 동작** | 신고 접수 → AI 분류·조사·검토 → 근거 재검증 → 정비팀 배정 화면 |
| **3** | **숫자로 확인** | 서비스 평가 200건 (dev 80 / test 120): 자동배정 정밀도, LLM2 부서 일치율, 시스템 오류 수 |
| **4** | **모를 때는 멈춤** | 근거가 없거나 검증에 실패하면 자동 배정하지 않고 사람 검토로. 배정 화면에 인용 근거 표시 |
| **5** | **나머지는 뒤로** | AWS 실배포(Terraform 정의까지), AI 워커 여러 개로 확장, 인원 현황의 배정 반영 |

<br>

</details>

## 어떻게 동작하나요?

<p align="center"><img src="assets/diagram-how-it-works-v2.png" width="1000" alt="동작 흐름: 접수, 기체 검증, LLM1 이력 비교, LLM2 독립 조사, LLM3 대조, 근거 재검증, 자동 배정 또는 사람 검토"></p>

1. **신고 접수** — 신고자가 기체번호·기종·증상을 입력합니다. 등록 정보와 다르면 백엔드가 맞는 값을 알려 줍니다.
2. **이력 비교 (LLM1)** — 증상만 보고 계통(ATA) 후보를 고른 뒤, 같은 기체의 해당 계통 수리이력과 비교합니다. 근거가 확실하면 바로 배정을 제안합니다.
3. **독립 조사 (LLM2)** — 이력으로 판단할 수 없으면, LLM1의 결론을 **보지 않은 채** SDR 사례와 Airbus 기술자료를 직접 검색해 판단합니다.
4. **검토 (LLM3)** — 두 판단과 원문을 대조합니다. 풀리지 않은 충돌이 있으면 다시 조사시킵니다(최대 2회).
5. **근거 재검증** — 백엔드가 인용 근거가 실제로 조회된 자료인지, 부서 코드가 맞는지 확인한 뒤 배정하거나 사람 검토로 보냅니다.
6. **정비팀 확인** — 배정된 정비팀이 수락하거나 거절하고, 관리자가 사람 검토 건을 처리합니다.

<details>
<summary><strong>요청 상태와 사용자별 유즈케이스 보기</strong></summary>

### 요청 상태

| 상태 | 언제? | 화면에서 |
|:--:|:--|:--|
| **QUEUED** | 접수 직후 | 처리 대기 |
| **CLASSIFYING → INVESTIGATING → REVIEWING** | AI가 단계별로 판단 중 | 3초마다 진행 단계 갱신 |
| **ASSIGNED** | 근거 검증 통과 | 담당 정비팀 대기 목록에 표시 |
| **HUMAN_REVIEW** | 근거 부족·판단 충돌·시간 초과·정비팀 거절 | 관리자 검토 목록에 표시 |
| **ACCEPTED** | 정비팀이 배정 수락 | 처리 이력에 기록 |

<br>

## 유즈케이스

로그인·회원가입처럼 모든 사용자에게 공통인 기능은 빼고, 세 액터의 업무만 담았습니다. UML 표기(연관 실선, «include», «extend»)만 쓰고, 업무 순서는 아래 처리 흐름에서 다룹니다.

<p align="center"><img src="assets/usecase-uml.svg" width="900" alt="MRO Agent 유즈케이스 다이어그램: 신고자, 정비팀, 관리자"></p>

| 액터 | 유즈케이스 | 화면 | API |
|:--|:--|:--|:--|
| **신고자** | 정비 신고 접수 | `/reporter` | `POST /api/v1/requests` |
| | 접수 정보 보정 | `/reporter` | 422 `NEEDS_SUPPLEMENT` 응답의 등록 값을 한 번에 적용 |
| | 처리 단계·결과 확인 | `/reporter` | `GET /api/v1/requests`, `GET /api/v1/requests/{id}/trace` |
| **정비팀** | 배정 요청·AI 판단 근거 확인 | `/maintenance` | `GET /api/v1/assignments`, `GET /api/v1/requests/{id}/trace` |
| | 배정 수락 | `/maintenance` | `POST /api/v1/assignments/{id}/accept` |
| | 배정 거절 (사유 입력) | `/maintenance` | `POST /api/v1/assignments/{id}/reject` → 사람 검토 `TEAM_REJECTED` |
| **관리자** | 사람 검토 처리 | `/admin` | `GET /api/v1/reviews`, `POST /api/v1/reviews/{id}/resolve` (`ASSIGN` 수동 배정 · `CLOSE` 종결) |
| | 전체 요청 조회 | `/admin` | `GET /api/v1/requests` |
| | 처리 통계 확인 | `/admin` | `GET /api/v1/stats/summary` (자동 배정률 등) |
| | RAG 데이터 상태 확인 | `/admin` | `GET /actuator/health` (ragData) |
| | 인원 현황 확인 | `/admin/staffing` | `GET /api/v1/staffing` |

- **AI 담당 부서 판단**은 신고 접수에 항상 포함되고(«include»), **자동 배정**은 근거 검증을 통과했을 때만 일어납니다(«extend»). 둘 다 사람이 아니라 시스템이 수행합니다.
- AI가 근거를 찾지 못하거나 정비팀이 배정을 거절하면, 요청은 관리자의 **사람 검토 처리** 목록으로 모입니다(처리 흐름: 실패 참고).

<br>

</details>

## 사용자가 마주하는 화면

<sub>2026-10-26 로컬 실행 화면입니다. 기존 시연·평가 데이터를 조회했으며, 접수 작성과 수동 배정 화면은 제출 전 상태입니다.</sub>

### ① 신고자: 접수와 진행 상황
기체번호와 증상을 적어 접수하고, 내 요청이 어느 단계에 있는지 실시간으로 봅니다.

<table>
<tr>
<td width="50%"><img src="assets/reporter-compose.png" alt="정비 신고 접수"></td>
<td width="50%"><img src="assets/reporter-detail.png" alt="요청 진행 단계"></td>
</tr>
</table>

### ② 정비팀: 배정 확인과 근거
우리 부서에 배정된 요청을 보고, AI 판단 요약·인용 근거·처리 이력을 확인한 뒤 수락하거나 거절합니다.

<table>
<tr>
<td width="50%"><img src="assets/maintenance-queue.png" alt="정비팀 배정 대기 목록"></td>
<td width="50%"><img src="assets/maintenance-evidence.png" alt="SDR 판단 근거와 별도 AMM 참고자료"></td>
</tr>
</table>

### ③ 관리자: 사람 검토와 운영 현황
사람 검토로 넘어온 요청을 수동 배정·종결하고, 자동 배정률과 RAG 데이터 상태, 부서별 인원 현황을 봅니다.

<table>
<tr>
<td width="50%"><img src="assets/admin-review.png" alt="사람 검토와 수동 배정"></td>
<td width="50%"><img src="assets/admin-staffing.png" alt="부서별 인원 현황"></td>
</tr>
</table>

<br>

## 얼마나 맞히나요?

실제 경로(backend → RabbitMQ → AI(gpt-5.4-mini) → MCP → PostgreSQL)로 **서비스 시나리오 200건**을 실행했습니다. (2026-10-23)

| 지표 | 결과 | 의미 |
|:--|:--:|:--|
| 자동배정 정밀도 | **96/145 (66.2%)** | AI가 자동 배정한 건 중 정답 부서 |
| LLM2 독립 조사 부서 일치 | **71/82 (86.6%)** | 이력 없이 사례 검색만으로 맞힌 비율 |
| 전체 평가 통과 | **97/200 (48.5%)** | 기대 상태·허용 부서·금지 이력 조건을 모두 만족 |
| 정답 부서를 맞히고도 보류 | **42건** | 확신이 부족해 사람 검토로 넘긴 경우 (성공에 합산하지 않음) |
| 시스템 오류 · 시간 초과 | **0건** | 200건 + 접수 검증 6건 전체 |

> 통과 기준을 엄격하게 잡았습니다. 정답 부서를 맞혀도 금지된 이력을 근거로 썼거나 사람 검토가 필요한 사례를 자동 배정하면 실패로 셉니다. 실제 항공사 운영 성능으로 일반화하지 않습니다.

<br>

<details>
<summary><strong>데이터 구성과 평가 방법 보기</strong></summary>

## 데이터 구성

AI가 근거로 찾는 **검색 데이터**와, 시연·평가에 쓰는 **업무 데이터** 두 가지입니다.

### 검색 데이터 (RAG)

| 데이터 | 규모 | 출처 | 쓰는 곳 |
|:--|--:|:--|:--|
| **SDR 사례** | 71,741건 | FAA 공개 정비 결함 보고 (2021~2026) | LLM2 사례 검색 (R02) |
| **SDR 임베딩** | 71,741건 | OpenAI text-embedding-3-large, 1024차원 | R02 단어 + 벡터 하이브리드 검색 |
| **근거 원문** | 107,137건 | SDR·AMM에서 인용 가능한 문장 (기술자료 조각은 별도) | 근거 재확인 (R05) |
| **AMM 참조** | 35,396건 | SDR에 인용된 정비 매뉴얼 번호 | 배정 후 참고 정보 (판단 근거로는 쓰지 않음) |
| **JASC 코드** | 548개 | FAA 결함 분류 코드 ↔ ATA 계통 | 결함 코드 후보 검색 (R01) |
| **기술자료** | 61편 (조각 496개) | Airbus Safety First | 기술자료 의미 검색 (R04) |

### 업무 데이터 (시드)

| 항목 | 규모 | 내용 |
|:--|--:|:--|
| 항공사 | 3 | 신고자 소속 항공사 |
| 기체 | 20 | 기체번호·기종 (A319·A320·A321, ceo·neo 포함) |
| 정비 부서 | 8 | 부서별 담당 ATA 계통 |
| 계정 | 15 | 신고자·정비팀·관리자 |
| 수리이력 | 1,600건 | 기체별 과거 수리 기록 (FAA SDR 2021~2026 기반으로 구성) |

<br>

## 평가 데이터 구성

전체 흐름을 보는 **서비스 통합 평가**, 입력 오류를 보는 **접수 검증**, 단계별 성능을 보는 **SDR 단위 평가** 세 가지로 나눴습니다.

### 1. 서비스 통합 평가 (200건)

실제 접수와 같은 입력(기체번호·항공사·기종·증상 4개 필드)만 보내고, 접수부터 배정까지 전체 경로를 평가합니다.

| 묶음 | 건수 | 무엇을 보나 |
|:--:|--:|:--|
| **A** | 64 | **같은 기체 반복 고장**: 수리이력을 근거로 바로 배정하는지 |
| **B** | 96 | **신규 증상**: 이력 없이 SDR 사례 검색만으로 맞히는지 |
| **C** | 16 | **다른 기체 유사 사례**: 다른 기체의 이력을 잘못 끌어오지 않는지 |
| **D** | 24 | **같은 ATA·다른 JASC 함정**: 계통은 같지만 담당이 다른 경우를 구분하는지 |

- 나눔: 개발용 dev 80건, 최종 확인용 test 120건. 기대 결과는 배정 197건, 사람 검토 3건입니다.
- 라벨 상태: 검토 전 161건, 최종 검토 필요 24건, 확정 15건. 평가 라벨과의 일치를 실제 정비 전문가 판정과 같다고 보지 않습니다.
- 정답(`expected`)과 출처 정보는 AI 입력에 넣지 않습니다.

**채점 기준**

| 항목 | 통과 조건 |
|:--|:--|
| 배정 기대 문항 | 상태 `ASSIGNED` + 허용 부서 일치 + 금지된 이력(`must_not_cite`)을 근거로 쓰지 않음, 셋 다 만족 |
| 사람 검토 기대 문항 | 상태 `HUMAN_REVIEW` |
| 자동배정 정밀도 | 통과한 자동배정 ÷ **전체 자동배정** (사람 검토가 기대됐는데 배정한 경우도 실패로 셈) |
| 평가 오류 | 추적 조회 실패·최종 판단 누락은 통과시키지 않고 오류로 따로 남김 |

### 2. 접수 검증 (6건)

존재하지 않는 기체, 맞지 않는 항공사·기종, 빠진 필수값, 권한 없는 접수를 보내 백엔드가 **422 응답으로 정확히 거절하는지** 확인합니다.

### 3. SDR 단위 평가 (검색·분류·부서 결정)

전체 흐름이 아니라 **검색 도구, LLM1 분류, LLM2 부서 결정을 하나씩 따로** 평가합니다.

| 평가셋 | 평가 대상 | 입력 | 정답 | 건수 |
|:--|:--|:--|:--|--:|
| 검색 | R01·R02 검색 도구 | SDR 원문에서 매뉴얼 참조를 지운 영어 문장 | JASC·ATA | 400 (부서별 50) |
| 분류 | LLM1 1단계: 증상만 보고 ATA 후보 고르기 | 증상만 남긴 한국어·영어 문장 | ATA | 200 (부서별 25) |
| 부서 결정 | LLM2: 근거를 받고 부서 고르기 | 증상 + 고정된 SDR 근거 5건 | 부서 | 200 (부서별 25) |
| JASC 변환 | JASC → 부서 변환 규칙 (회귀 테스트) | JASC 코드 | 부서 | 548 (전체 코드) |

- **정답 누수 방지:** 검색 대상은 2021~2026년 SDR이고, 평가셋은 **2019~2020년** SDR로 만들었습니다. 신고 번호나 원문이 같은 것도 한 번 더 뺐습니다.
- **부서별 균등 추출:** SDR의 절반 이상이 구조(STR) 계통이라, 고르게 뽑지 않으면 STR만 맞혀도 점수가 높게 나옵니다. 그래서 부서 8개마다 같은 수를 뽑았습니다.
- **정답 라벨:** 신고자가 붙인 JASC 코드 → ATA(앞 두 자리) → 담당 부서. 예: JASC `3231` → ATA `32` → 유압(HYD).
- 세 평가셋은 서로 겹치지 않습니다.

<br>

</details>

## 리포지토리 구성

| 리포지토리 | 한 줄 설명 | 주요 기술 |
|:--|:--|:--|
| [**dashboard**](https://github.com/mro-agent-proj/dashboard) | 신고자·정비팀·관리자 화면 | Vue 3 · TypeScript · PrimeVue 4 |
| [**backend**](https://github.com/mro-agent-proj/backend) | 계정·요청·배정 API, DB·데이터 적재, MCP 토큰 발급 | Java 21 · Spring Boot · Flyway |
| [**LLM**](https://github.com/mro-agent-proj/LLM) | LLM1 분류 → LLM2 조사 → LLM3 검토 AI 서버 | Python · FastAPI · LangGraph · LangChain |
| [**agent**](https://github.com/mro-agent-proj/agent) | 업무 조회·근거 검색 MCP 서버 2개 (도구 9개) | Python · MCP · pgvector |
| [**integration**](https://github.com/mro-agent-proj/integration) | 서비스 4개의 검증된 커밋 조합, 실행·평가 스크립트, 모니터링 | Docker Compose · GitHub Actions · Prometheus · Grafana |
| [**infra**](https://github.com/mro-agent-proj/infra) | AWS 인프라와 운영 배포 | Terraform · ECS Fargate · WAF |
| [**deliverables**](https://github.com/mro-agent-proj/deliverables) | 최종 결과보고서, 아키텍처 설계서·다이어그램 | Markdown · Mermaid |

<br>

## 기술 스택

<div align="center">

**Backend**<br>
![Java](https://img.shields.io/badge/Java_21-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_16-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?style=flat-square&logo=rabbitmq&logoColor=white)
![Flyway](https://img.shields.io/badge/Flyway-CC0200?style=flat-square&logo=flyway&logoColor=white)

**AI & RAG**<br>
![Python](https://img.shields.io/badge/Python_3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-1C3C3C?style=flat-square&logo=langchain&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=flat-square&logo=openai&logoColor=white)
![pgvector](https://img.shields.io/badge/pgvector-336791?style=flat-square&logo=postgresql&logoColor=white)
![MCP](https://img.shields.io/badge/MCP-000000?style=flat-square)

**Frontend**<br>
![Vue.js](https://img.shields.io/badge/Vue_3-4FC08D?style=flat-square&logo=vuedotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![PrimeVue](https://img.shields.io/badge/PrimeVue_4-41B883?style=flat-square)
![Vite](https://img.shields.io/badge/Vite-646CFF?style=flat-square&logo=vite&logoColor=white)

**Infra & DevOps**<br>
![Terraform](https://img.shields.io/badge/Terraform-844FBA?style=flat-square&logo=terraform&logoColor=white)
![Amazon ECS](https://img.shields.io/badge/ECS_Fargate-FF9900?style=flat-square&logo=amazonecs&logoColor=white)
![Amazon RDS](https://img.shields.io/badge/RDS-527FFF?style=flat-square&logo=amazonrds&logoColor=white)
![CloudFront](https://img.shields.io/badge/CloudFront_·_WAF-8C4FFF?style=flat-square&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat-square&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white)

</div>

<br>

## 기술 문서

필요한 항목을 펼쳐 상세 설계와 구현 내용을 확인할 수 있습니다.

<details>
<summary><strong>시스템 아키텍처와 실행 제한</strong></summary>

## 시스템 구조

> 여기부터는 개발자를 위한 내용입니다.

<p align="center"><img src="assets/diagram-system-architecture-v2.png" width="1000" alt="구성도: dashboard, backend, RabbitMQ, AI 서버, MCP 서버, PostgreSQL"></p>

> **Harness** — MCP 도구 10초·재시도 1회 · 모델 호출 45초 · 단계별 도구 호출 8회 · 재조사 최대 2회 · AI 실행 280초 · 백엔드 처리 한도 5분 · 실행별 MCP 토큰 10분 · LLM 출력 스키마 검증 실패 시 1회 재요청

<br>

</details>

<details>
<summary><strong>정상·실패 처리 흐름</strong></summary>

## 처리 흐름: 정상

<p align="center"><img src="assets/diagram-sequence-normal.png" width="1000" alt="정상 처리 시퀀스: 접수부터 자동 배정과 정비팀 수락까지"></p>

<br>

## 처리 흐름: 실패

<p align="center"><img src="assets/diagram-sequence-failure.png" width="1000" alt="실패 처리 시퀀스: 접수 보완, 재조사 한도, 근거 재검증 실패, AI 실패·시간 초과"></p>

| 실패 | 어디서 막나 | 결과 |
|:--|:--|:--|
| 접수 정보 불일치 · 필수값 누락 · 권한 없음 | `IntakeValidator.validate` | 422, AI 실행 안 함 |
| 이력·사례로 근거를 찾지 못함 | LangGraph `hold` | 사람 검토 `AGENT_NEEDS_HUMAN` |
| 재조사 2회로도 충돌이 남음 | LangGraph `limit` → `hold` | 사람 검토 `RETRY_LIMIT` |
| 인용 근거가 실제 조회 기록과 다름 | `EvidenceValidator.findProblems` | 사람 검토 `VALIDATION_FAILED` |
| 모델·도구 오류, 5분 초과, 메시지 처리 실패 | `FAILED` 이벤트, `AgentTimeoutJob`, `RunDeadLetterListener` | 사람 검토 `SYSTEM_ERROR` |
| 정비팀이 배정을 거절 | `AssignmentService.reject` | 사람 검토 `TEAM_REJECTED` |

<br>

</details>

<details>
<summary><strong>LangGraph: AI 판단 흐름과 노드 역할</strong></summary>

## LangGraph: AI 판단 흐름

AI 서버는 LangGraph로 판단 단계를 그래프로 연결합니다. 단계마다 상태(State)를 체크포인트에 저장해, 서버가 재시작돼도 멈춘 단계부터 이어 갑니다.

<p align="center"><img src="assets/diagram-langgraph-flowchart-v2.png" width="600" alt="LangGraph AI 판단 순서도: classify, investigate, review, finalize, hold"></p>

| 노드 | 하는 일 |
|:--|:--|
| `classify` | 증상만 보고 ATA 후보를 고른 뒤, 같은 기체의 수리이력과 비교 (LLM1) |
| `investigate` | LLM1 결론·수리이력을 받지 않고, 원본 신고와 자체 검색 결과만으로 조사 (LLM2) |
| `merge_for_review` | LLM1·LLM2 결과를 합치기 전에 요청·버전·검색 범위·스냅샷이 그대로인지 검사 |
| `review` | 두 판단과 원문을 대조해 통과·재조사·보류를 결정 (LLM3) |
| `finalize` | 인용 근거를 다시 조회해 확인하고, 근거 SDR의 참고 AMM 번호를 첨부 |
| `hold` | 근거가 부족하거나 재조사 한도를 넘으면 사람 검토로 보냄 |

<br>

</details>

<details>
<summary><strong>MCP 서버별 도구와 호출 주체</strong></summary>

## MCP 도구

AI는 MCP(Model Context Protocol)로 두 서버의 도구 9개를 호출합니다. 두 서버 모두 **읽기 전용**이며, 요청마다 백엔드가 발급한 토큰과 검색 범위 안에서만 조회합니다.

| 서버 | ID | 도구 | 하는 일 | 누가 부르나 |
|:--|:--:|:--|:--|:--|
| business-mcp | B01 | `get_request_context` | 접수 내용과 이 실행에 등록된 검색 범위 | 실행기 (시작·최종 확정) |
| | B02 | `get_aircraft_history` | 같은 기체의 이전 수리이력 (접수 시각 이전만) | LLM1, LLM3, 최종 검증 (LLM2는 차단) |
| | B03 | `search_departments` | 정비 부서 8개와 담당 ATA | 실행기 (단계마다 고정 확인) |
| rag-mcp | R01 | `search_jasc_codes` | JASC 결함 코드 후보와 정의 | LLM2, LLM3 |
| | R02 | `search_sdr_cases` | SDR 사례 71,741건 하이브리드 검색 (단어 + 벡터, 한국어 질의 가능) | LLM2, LLM3 |
| | R03 | `search_amm_references` | AMM 번호로 그 번호가 인용된 SDR 문장 조회 | LLM 도구 목록에서 제외 |
| | R04 | `search_technical_docs` | Airbus Safety First 기술자료 의미 검색 | LLM2, LLM3 |
| | R05 | `get_evidence` | 받은 근거의 원문과 인용 위치 재확인 | LLM2, LLM3, 최종 검증 |
| | 확장 | `get_sdr_amm_references` | 근거 SDR에 인용된 AMM 번호 조회 (참고용 첨부) | 실행기 (배정 확정 후) |

> 서비스 시나리오 200건 실행에서 LLM2는 R01을 77회, R02를 84회 골라 썼고, 최종 검증은 R05를 97회 호출했습니다.

<br>

</details>

<details>
<summary><strong>판단 검증·보안·배포 구성</strong></summary>

## 기술적으로 신경 쓴 점

<table>
<tr>
<td width="50%" valign="top">

### AI 판단의 신뢰성
- **판단 분리** — LLM2는 LLM1의 결론·수리이력을 전달받지 않고 원본 신고와 자체 검색 결과만으로 독립 판단합니다. 같은 결론을 베끼는 확증 편향을 막습니다.
- **근거는 코드가 검증** — 인용한 근거가 실제로 조회됐는지, 발췌가 원문에 있는지, 부서 코드가 유효한지 AI 서버와 백엔드가 두 번 확인합니다.
- **출력 검증** — 모든 LLM 출력을 pydantic으로 검증하고, 실패하면 오류를 알려 주고 한 번 더 받습니다.
- **AMM은 근거에서 제외** — 정비 매뉴얼 번호는 배정이 끝난 뒤 참고 정보로만 붙입니다.

</td>
<td width="50%" valign="top">

### 보안 · 운영
- **읽기 전용 MCP** — MCP 서버는 SELECT 전용 DB 계정으로만 접속하고, 쓰기 권한이 있으면 기동을 거부합니다.
- **실행별 위임 토큰** — 백엔드가 요청마다 검색 범위를 등록하고 수명이 짧은 MCP 토큰 2개를 발급합니다.
- **복구 가능한 실행** — RabbitMQ DLQ, 5분 처리 한도(넘으면 사람 검토), 단계별 체크포인트로 재시작해도 이어서 처리합니다.
- **검증된 조합만 배포** — 서비스 main이 바뀌면 봇이 통합 검증을 돌리고, 통과한 조합만 infra에서 승인 후 배포합니다.

</td>
</tr>
</table>

<br>

## 배포 구성

<p align="center"><img src="assets/diagram-deployment-v2.png" width="1000" alt="배포 흐름: 서비스 PR, CI·병합, integration 통합 검증, verified 태그, 승인 후 terraform apply"></p>

- 코드는 **서비스 레포에서만** 바꿉니다. integration은 봇이 자동으로 갱신하고, 운영 배포는 **infra에서만** 합니다.
- GitHub Actions는 **OIDC**로 AWS에 접근합니다. 장기 액세스 키를 쓰지 않습니다.
- CloudFront 앞의 **WAF**가 관리형 규칙, SQL 인젝션 차단, 로그인 요청 횟수 제한을 맡습니다.

AWS 구성은 [infra](https://github.com/mro-agent-proj/infra) 레포의 Terraform으로 정의했습니다.

<p align="center"><img src="assets/aws-architecture.png" width="860" alt="AWS 아키텍처"></p>

<br>

</details>

## 용어 사전

<details>
<summary>항공 정비가 처음이라면 펼쳐 보세요</summary>
<br>

| 용어 | 쉬운 설명 |
|:--|:--|
| **MRO** | Maintenance, Repair & Overhaul. 항공기 정비·수리·분해점검 |
| **정비 신고** | 조종사·정비사가 발견한 결함을 알리는 요청서 |
| **ATA 챕터** | 항공기 계통 분류 번호. 예: 21 공조, 32 착륙장치, 72 엔진 |
| **SDR** | Service Difficulty Report. FAA에 보고된 정비 결함 사례 |
| **JASC 코드** | SDR에서 쓰는 결함 분류 코드. ATA 계통과 대응됩니다 |
| **AMM** | Aircraft Maintenance Manual. 기종별 정비 매뉴얼 |
| **RAG** | AI가 답하기 전에 관련 자료를 먼저 검색해 참고하는 방식 |
| **MCP** | Model Context Protocol. AI가 외부 도구·데이터를 호출하는 표준 규약 |
| **LangGraph** | AI 판단 단계를 그래프로 연결해 흐름을 제어하는 프레임워크 |
| **사람 검토** | AI가 확신하지 못한 요청을 관리자가 직접 판단하는 단계 |

</details>

<br>

## 팀

<table>
<tr>
<td align="center" width="16%"><a href="https://github.com/younghoyk"><img src="https://github.com/younghoyk.png" width="64" alt="younghoyk"><br><sub>@younghoyk</sub></a></td>
<td align="center" width="16%"><a href="https://github.com/Iod-oxy"><img src="https://github.com/Iod-oxy.png" width="64" alt="Iod-oxy"><br><sub>@Iod-oxy</sub></a></td>
<td align="center" width="16%"><a href="https://github.com/kkwakna"><img src="https://github.com/kkwakna.png" width="64" alt="kkwakna"><br><sub>@kkwakna</sub></a></td>
<td align="center" width="16%"><a href="https://github.com/ybkeem"><img src="https://github.com/ybkeem.png" width="64" alt="ybkeem"><br><sub>@ybkeem</sub></a></td>
<td align="center" width="16%"><a href="https://github.com/goldmirr"><img src="https://github.com/goldmirr.png" width="64" alt="goldmirr"><br><sub>@goldmirr</sub></a></td>
<td align="center" width="16%"><a href="https://github.com/Weeny326"><img src="https://github.com/Weeny326.png" width="64" alt="Weeny326"><br><sub>@Weeny326</sub></a></td>
</tr>
</table>

<br>

<div align="center">

**AI가 찾고, 근거가 증명하고, 사람이 결정합니다.**

<sub>© 2026 MRO Agent</sub>

</div>
