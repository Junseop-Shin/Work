# S06 — 도메인 온톨로지 구축 사례 ②: DTC (Digital Twin Construction)

> 2026-08-03 | 온톨로지, DigitalTwin, DTC, BOT, BEO, SOSA, DCAT, GeoSPARQL, SHACL, RDF파이프라인, ConSLAM

> 📖 [강의 목차/진행](README.md) · 이전: [S05B MANUMATE](S05B-MANUMATE-제조-상호운용성.md) · 다음: S07 구조적·품질 평가 (OQuaRE)
>
> 📎 부록: [S05-06-1 세 논문을 잇는 파이프라인](S05-06-1-세-논문을-잇는-파이프라인.md)

## 한 줄 요약

건설 현장의 **디지털 트윈(DTC)** 플랫폼을 위한 참조 아키텍처와 온톨로지 프레임워크 제안.
설계의 축은 셋이다 — **계획(PII)과 실제 관측(PSI)을 분리**하고,
**Data / Information / Knowledge 세 층에 각기 다른 저장소**를 쓰고,
**AI 서비스의 출력을 구조화된 KG 업데이트로** 받는다.
그리고 이 논문의 가장 실용적인 결정은 **"모든 데이터를 RDF에 넣지 않는다"**는 것 —
포인트클라우드는 오브젝트 스토리지에, 센서 스트림은 시계열 DB에 두고
**RDF 그래프는 "무엇이 어디에 있고 무엇과 연결되는가"를 아는 semantic index** 역할만 한다.

---

# 01 Intro

## 1. 논문 개요와 발표 방향

- **건설 현장 업무를 돕는 디지털 트윈(Digital Twin) 구성을 위한 온톨로지 & 지식그래프 제안**
- 건설 현장에서 발생하는 데이터들의 **종류와 도메인 특성을 고려**해
  적합한 온톨로지를 도입하고 지식그래프 구조를 설계
- 도메인 특성상 익숙한 형태의 연구 논문보다는 **technical report나 use case에 대한 보고서 같은 내용**

> 논문 특성상, 추상적인 규칙이나 일반적인 설계 원칙보다는
> **구체적인 구현 내용과 사례를 주로 다룰 예정**

| | |
|---|---|
| 제목 | *Reference architecture and ontology framework for digital twin construction* |
| 저자 | Jonas Schlenger, Kacper Pluta, Alwyn Mathew, Timson Yeung, Rafael Sacks, André Borrmann |
| 소속 | TUM(뮌헨공대) · Technion · Université Gustave Eiffel/CNRS · Inria · University of Cambridge |
| 출처 | *Automation in Construction* 174 (2025) 106111 · Elsevier |
| 키워드 | Digital Twin Construction(DTC) · Ontology · Reference architecture · Building construction · Data management · Linked building data(LBD) · Data integration · Interoperability · Digital twin(DT) |

## 2. Rethinking — 온톨로지 돌아보기

- 처음 접하는 입장에서 **추상적인 표현과 설계원칙 나열**로 인해 복잡하고 방대하게 느껴지기도 하나,
- 사실은 결국 **도메인 데이터를 적절히 유형화, 형식화, 연결, 저장하여 잘 활용하고자 하는 것**

**What is ontology?** — `인공지능은 인간을 돕는다`

| 층 | 하는 일 |
|---|---|
| **INFERENCE** | 추론 |
| **ONTOLOGY** | Node(AI · Human) · relation(helps) · Attribute |
| **DATA** | 원시 데이터 |

## 3. 결과물 미리보기

**건물 구조 클래스 (Fig. 8 UML)**

| 구분 | 클래스 · 관계 |
|---|---|
| **공간 계층** (포함 관계) | `bot.Site` →`bot.hasBuilding`→ `bot.Building` →`bot.hasStorey`→ `bot.Storey` →`bot.hasSpace`→ `bot.Space` |
| **부재 계층** (상속) | `beo.BuildingElement` ← `beo.Column` · `beo.Door` · `beo.Wall` · `beo.Slab` · … |
| **공간 ↔ 부재** | `bot.Zone` —`bot.hasElement`→ `bot.Element` |
| **부재 간** | `bot.hasSubElement` · `bot.adjacentElement` |
| **기하** | `bot.Element` —`dtc.hasBoundingbox`→ `geo.Geometry`(`geo.asWKT`) ← `geo.SpatialObject` |

**논문이 제안하는 DTC ontology가 붙는 자리** — `bot.Element`에 네 속성이 추가된다:
`dtc.id` · `dtc.IsAsDesigned` · `dtc.progress` · `dtc.timeStamp`

| 슬라이드가 표시한 것 | 의미 |
|---|---|
| **포함 관계** | "site_1은 building_1을 포함함" |
| **온톨로지** | `beo:` = 건축 부재 |
| **클래스** | `Wall` = 벽 |
| **상속** | `beo.Column`은 `beo.BuildingElement`의 하위 클래스 |

**센서 시스템 (Fig. 13, SOSA/SSN)**

```turtle
inst:monitoring_pltf_83
    rdf:type sosa:Platform ;
    sosa:hosts inst:camera_233 ;
    sosa:hosts inst:camera_235 ;
    sosa:hosts inst:laserscanner_252 .

inst:laserscanner_252
    rdf:type sosa:Sensor ;
    sosa:madeObservation inst:observation_252_008 ;
    sosa:madeObservation inst:observation_252_009 ;
    sosa:madeObservation inst:observation_252_010 .
```

---

# 02 Preliminaries

## 4. 건설 현장 관련 정보 — PII와 PSI

**이 논문 전체를 관통하는 구분이다.**

| | 정의 | 예 |
|---|---|---|
| **PII** (Project **Intent** Information) | 프로젝트에 대한 **설계 · 시공 계획** 정보 | ① 필요 자원: 작업반 2개, 크레인 1대, 자재 30개 · ② 스케줄: column work package 수행 기간 03/01–03/15 |
| **PSI** (Project **Status** Information) | **실제 현장** 정보 | ① 현장에 자원이 실제로 얼마나 있는지 · ② 작업이 실제로 언제 끝났는지 |

**데이터 종류**

- 건설 현장을 **레이저 스캐너로 측정한 point cloud**
- 인적 정보, 자재 정보
- **KPI 시계열 데이터**
- 센서로 측정된 시계열 데이터
- 기타 이미지, 모니터링 데이터

## 5. 디지털 트윈 (Digital Twin)

> **실제 대상/시스템을 디지털 공간에 표현하고, 실제 상태 변화가 디지털 모델에 계속 반영되도록 만든 시스템**
> 단순한 3D 모델이 아니라, 건설 현장의 계획/상태 정보를 관리하기 위한 **종합적인 데이터 관리 플랫폼**

| 구성 | 내용 |
|---|---|
| **Physical object / system** | 실제 건물, 공장, 로봇, 항공기, 건설 현장 |
| **Digital representation** | 3D 모델, 데이터 모델, 시뮬레이션 모델, **지식그래프** |
| **Data connection** | 센서, 이미지, 포인트클라우드, IoT, 작업 기록 등을 통한 업데이트 |

## 6. 논문 주제와 DT 6층 아키텍처

- **논문 주제 요약**: DTC(Digital Twin Construction) 플랫폼의 **참조 아키텍처와 온톨로지 프레임워크** 제안
- 건설 공사 절차는 **[계획 → 시공 → 관측 → 비교]의 반복**
  ⇒ **이 사이클의 데이터를 관리하는 것이 Digital Twin의 목적**

**Six-layer DT architecture** *(Fig. 2, Mostafa et al.)*

| 층 (위 → 아래) | 하는 일 |
|---|---|
| **Consumption Layer** | data visualization and reporting |
| **Service Layer** | providing data service and API |
| **Inference Layer** | arithmetic and AI-based algorithms |
| **Persistence Layer** | DBs with access control and high-performance file systems |
| **Ingestion Layer** | data ingestion & conversion |
| **Physical Layer** | sensors, machines, people |

**Digital Twin ↔ Physical Twin 사이클**

| 방향 | 흐름 |
|---|---|
| Digital → Physical | `PII` → **Build** |
| Physical → Digital | **Monitor** → `Raw Data` → **Process** → `PSI` |
| 내부 | `PII` ↔ **Compare** ↔ `PSI` → **Feedback** → Build |
| 보존 | `PSI` → **Archive** → Historical DTs |

## 7. RDF graph와 Ontology

**RDF graph** (Resource Description Framework graph)

- **[주어 – 관계 – 목적어]** 형태의 문장들로 개념을 연결하는 그래프
- 즉, **"A와 B가 어떤 관계인지"**를 엮어서 구성하는 지식 연결망

**건설 현장 데이터 적용 예시**

| 주어 | 관계 | 목적어 | 자연어 |
|---|---|---|---|
| `building_1` | `hasStorey` | `storey_1` | building_1은 storey_1을 가진다 |
| `storey_1` | `hasSpace` | `space_101` | storey_1은 space_101을 가진다 |
| `space_101` | `hasElement` | `column_415` | space_101은 column_415를 포함한다 |
| `column_415` | `type` | `Column` | column_415는 Column 타입이다 |

**Ontology**

- **RDF graph에서 사용할 클래스와 관계의 규칙**
- e.g. `bot:Building`, `bot:hasStorey`, `beo:Column`
- 간단히 생각하면, **여러 정보들을 적절히 카테고리화해서 사용하는 네이밍 규칙일 뿐임**

---

# 03 Ontology Design

## 8. 도메인 특성을 고려한 설계

**이 표가 논문 설계 결정의 근거 전부다.**

| 건설 도메인의 특성 | 온톨로지·지식그래프 설계 방향 |
|---|---|
| 계획과 실제가 항상 일치하지는 않음 | → **계획(PII)과 실제 상태(PSI)를 분리** |
| 실제 작업이 계획과 1:1로 대응하지 않음 | → **intent–status 대응 관계를 명시적으로 모델링** |
| 센서 데이터가 대용량 · 비정형 | → **KG에는 메타데이터만, 원본은 외부 저장소에** |
| 작업 단위의 granularity가 다양함 | → **WorkPackage – Activity – Task** 프로세스 계층 |
| AI 서비스가 결과를 계속 추가함 | → **스키마 검증(SHACL)과 이력·출처 관리** |
| 현장과 요구사항이 계속 변함 | → **확장 가능한 온톨로지 + API 기반 업데이트** |

> **왼쪽 칸을 제조로 바꿔 읽으면 그대로 우리 과제다.**
> "계획과 실제가 일치하지 않음"은 표준작업지침 vs 실제 작업 로그이고,
> "센서 데이터가 대용량·비정형"은 설비 로그이고,
> "작업 단위 granularity"는 Lot–공정–스텝이다.
> [S05B MANUMATE](S05B-MANUMATE-제조-상호운용성.md)에는 **계획/실제 축이 아예 없었다** —
> 값 하나가 참이라고 전제했다. 이 논문은 그 축을 데이터 모델의 최상위 구분으로 올린다.

## 9. 사용한 온톨로지 — 기존 재사용 + 신규 하나

> **대부분 기존 온톨로지들을 따라서 사용하되, Digital Twin Construction(DTC)을 위한 신규 온톨로지 제안**

| 온톨로지 | 담당 영역 |
|---|---|
| **BOT / BEO** | 건물 구조: 공간(zone)·부재의 위상 관계와 분류 |
| **SOSA / SSN** | 센서 시스템, 관측(Observation), 측정 절차 |
| **DCAT** | 외부에 저장된 데이터셋의 카탈로그: 위치, 포맷, 배포 정보 |
| **GeoSPARQL / FOG** | 기하 참조: WKT bounding box, 외부 geometry 파일 링크 |
| **DTC Ontology** *(신규 정의)* | **Digital Twin 특성에 맞는 여러 활용을 위해 새롭게 제안** |

> [S04 6.2절](S04-온톨로지-엔지니어링-방법론.md) Kickoff의 "재사용 가능한 기존 온톨로지를 탐색"이
> 실제로는 이렇게 생겼다. **네 벌을 그대로 가져다 쓰고, 없는 것 하나만 새로 만든다.**
> 그리고 `DCAT`이 들어 있는 게 이 설계의 열쇠다 — 11절에서 그 이유가 나온다.

## 10. Three-Layer Architecture

| 층 | 담는 것 | 예 |
|---|---|---|
| **Knowledge Layer** | KPI, 지연, 생산성 등 **의사결정용 인사이트** | 일정 준수율, work package별 결함 수 |
| ↑ | *집계 · 비교 → 지식 도출* | |
| **Information Layer** | **해석된 엔티티**: 부재, 공정, 자원 — **PII와 PSI가 여기서 연결됨** | RDF graph로 표현되는 의미 계층 |
| ↑ | *해석 (interpretation)* | |
| **Data Layer** | **원시 관측**: point cloud, 이미지, 시계열 센서 스트림 | object storage, 시계열 DB 등 전용 저장소 |

**설계 원칙**

- Raw data를 해석해서 **유의미한 정보로 변환**하기 위한 구조
- **위로 갈수록 볼륨은 줄고 의미는 커짐**
- 저장/질의 요건이 달라 **하나의 DB로는 모두에 최적일 수 없음**
- 따라서 **층별로 알맞은 저장소를 쓰되, 접근은 공통 API 뒤로 숨김**
- **AI 서비스는 주로 Data를 읽고, Information/Knowledge를 작성**

### 10.1 Data Layer — Raw monitoring data

| | 내용 |
|---|---|
| **데이터 예시** | Point cloud scan · Site images/videos · Sensor time-series · GPS/equipment location logs |
| **저장 방식** | Point cloud·image → **Object storage** / Sensor values·GPS logs → **Time-series DB** / **Metadata → RDF catalog graph** |

```turtle
inst:observation_252_009
    rdf:type sosa:Observation ;
    sosa:madeBySensor inst:laserscanner_252 ;
    sosa:hasResult inst:scan_dataset_252_009 .

inst:scan_dataset_252_009
    dcat:distribution inst:scan_file_252_009 .

inst:scan_file_252_009
    dcat:downloadURL "minio://pointclouds/scan_252_009.ply" .
```

> **그래프에는 `minio://` 경로만 있다.** 실제 PLY 파일은 그래프 밖에 있다.

### 10.2 Information Layer — Interpreted site state

| | 내용 |
|---|---|
| **데이터 예시** | As-designed building element · As-built building element · Planned task · Performed action |
| **저장 방식** | Main representation → **RDF information graph** / Detailed geometry → **Object storage** / Geometry link·bounding box → **RDF graph** |

```turtle
inst:column_415
    rdf:type beo:Column ;
    dtc:isAsDesigned true .

inst:task_column_415
    rdf:type dtc:Task ;
    dtc:hasTarget inst:column_415 ;
    dtc:plannedEnd "2022-03-15" .
```

### 10.3 Knowledge Layer — Derived insight / KPI

| | 내용 |
|---|---|
| **데이터 예시** | Delay KPI · Cycle time · Throughput · Defect count · Productivity indicator |
| **저장 방식** | KPI definition/metadata → **RDF catalog graph** / KPI values over time → **Time-series DB** |

| 저장소 | 담는 것 |
|---|---|
| RDF graph | `delay_ratio_kpi`는 work package의 지연 정도를 나타내는 KPI이다 (**정의**) |
| Time-series DB | `2022-03-29, WP_column_f1, delay_ratio = 0.42` (**값**) |

> **KPI도 정의와 값을 갈라 놓는다.** 정의는 그래프에, 값은 시계열 DB에.
> 이 논문이 층마다 반복하는 패턴이다 — **의미는 그래프, 부피는 밖.**

## 11. RDF Graph as Semantic Backbone

> **모든 데이터를 RDF에 넣지 않음**: RDF 그래프는 **"무엇이 어디에 있고, 무엇과 연결되는가"를 아는 semantic index** 역할

- 대용량 파일(point cloud, 이미지)은 **object storage**에, 고빈도 센서 스트림은 **시계열 DB**에 저장
- Knowledge Graph에는 **메타데이터(DCAT)와 참조 링크만** 저장하고,
  서비스는 Knowledge Graph에서 **위치를 찾아 원본에 접근**

| 저장소 | 담는 것 | 사례 |
|---|---|---|
| **Object Storage** | point cloud(PLY), 이미지 | MinIO |
| **RDF Graph** | **엔티티 · 관계 · 메타데이터** | GraphDB |
| **Time-series DB** | 센서 스트림, 계측값 | InfluxDB |

- 위쪽에 **AI 서비스 · 대시보드**가 붙어 **SPARQL · API로 질의**
- 그래프와 두 저장소 사이는 **파일 경로 · 데이터셋 메타데이터로만 참조** (원본은 그래프 밖)

> **Knowledge Graph는 데이터베이스가 아니라 매뉴얼에 가까움 ("의미의 지도" 역할)**

> **이 문장이 이 회차에서 가장 실무적이다.** 우리 파이프라인은 GraphDB를 적재 대상으로 놓고
> "무엇까지 그래프에 넣을 것인가"를 정하지 않았다. 이 논문의 답은 명확하다 —
> **부피가 큰 것은 넣지 않고 위치만 넣는다.** 그리고 그 위치를 표현하는 표준이 `DCAT`이다.
> [S05-06-1 4.3절](S05-06-1-세-논문을-잇는-파이프라인.md)에서 이 결정을 파이프라인에 얹는다.
> 다만 **Wikidata는 정반대로 49억 트리플을 전부 그래프에 넣는다** — 두 논문이 갈리는 지점이다.

## 12. 프로세스 클래스 (Fig. 7 — project intent side)

**계획(PII) 쪽 프로세스 모델이다.**

| 클래스 | 속성 |
|---|---|
| `dtc.ConstructionSchedule` | `dtc.id` · `baselinePlanFrom` · `baselinePlanTill` |
| `dtc.Process` (상위) | `dtc.id` · `startTime` · `endTime` · `classificationSystem` · `classificationCode` |
| `dtc.WorkPackage` · `dtc.Activity` · `dtc.Task` | `dtc.Process`를 상속 · Task는 `dtc.contractor` 추가 |
| `dtc.ResourceAssignment` | `dtc.id` · `startTime` · `endTime` · `quantity` · `utilizationRate` |
| `dtc.Precondition` | `dtc.id` · `dtc.fulfilled` |

**프로세스 계층 (합성 관계)**

| 상위 | 관계 | 하위 |
|---|---|---|
| `ConstructionSchedule` | `dtc.hasWorkPackage` | `WorkPackage` |
| `WorkPackage` | `dtc.hasActivity` | `Activity` |
| `Activity` | `dtc.hasTask` | `Task` |

**Precondition 4종 — 작업이 시작되려면 무엇이 충족돼야 하는가**

| 하위 클래스 | 추가 속성 |
|---|---|
| `dtc.ProcessPrecondition` | (선행 공정) · `dtc.hasSequenceType` → `dtc.SequenceType` |
| `dtc.ZonePrecondition` | `dtc.availableFrom` · `dtc.availableTill` |
| `dtc.InformationPrecondition` | — |
| `dtc.ExternalFactorPrecondition` | `dtc.thresholdValue` |

**그 외 관계**

| 관계 | 잇는 것 |
|---|---|
| `dtc.hasTarget` | `Task` → `bot.Element` |
| `dtc.hasResourceAssignment` | `Activity` · `Task` → `ResourceAssignment` |
| `dtc.hasPrecondition` | `Process` → `Precondition` |
| `dtc.isPerformedIn` · `dtc.requiresZone` | → `dtc.AsPlannedWorkingZone` |
| `b2t.requiresProcess` | `bot.Element` → `Process` |

> **Precondition을 클래스로 올린 게 눈에 띈다.** "선행 공정 · 구역 확보 · 정보 확보 · 외부 요인"
> 네 가지를 각각 클래스로 두고 `fulfilled` 플래그를 붙였다.
> 제조로 옮기면 **공정 투입 전 조건**(선행 공정 완료 · 설비 가용 · 레시피 확정 · 온습도)이 그대로 대응한다.

## 13. RDF 인스턴스 예시

**기하 표현 (Fig. 12)** — 같은 벽에 **인라인 좌표**와 **외부 파일 링크**를 둘 다 건다.

```turtle
inst:wall_415
    rdf:type beo:Wall ;
    geo:hasGeometry geo:geometry_415 .

geo:geometry_415
    rdf:type geo:Geometry ;
    geo:asWKT "POLYHEDRALSURFACE(((12.1 13.7 -3.0, 12.1 9.3 -3.0, 14.9 9.3 -3.0, 12.1 13.7 -3.0)), … )"^^xsd:string ;
    fog:asPLY "https://dtc-ontology.de/wall415.ply"^^xsd:string .
```

> `geo:asWKT`는 그래프 안에 좌표를 직접 넣고, `fog:asPLY`는 외부 파일을 가리킨다.
> **간단한 형상은 안에, 상세 메시는 밖에** — 11절 원칙의 구체적 적용례다.

## 14. DTC Platform 아키텍처

**RDF graph와 Digital Twin 기반 플랫폼 아키텍처**

- Raw data들을 저장하는 DB
- 서버와의 **Web API 통신**
- **RDF graph 기반으로 동작하는 digital twin**
- Digital twin 정보를 보여주는 **대시보드**

**층별 저장소 배치** *(Fig. 16 관련)*

| 층 | 그래프 | 붙는 저장소 |
|---|---|---|
| **Knowledge** | Catalogue graph | Time-series DB (KPIs) |
| **Information** | RDF graph | Object Storage (Element geometry) |
| **Data** | Catalogue graph | Time-series DB (Sensor data) · Object Storage (Point clouds, images) · Native DB (other data) |

**플랫폼 주변**

| 구성 | 역할 |
|---|---|
| **Data validation** | 들어오는 데이터 검증 |
| **Authentication** | 인증 |
| **Translation and Injection** | Project Intent Data · Raw Monitoring Data를 받아 변환·주입 |
| **Service 1~X (DTS)** | 외부 서비스가 Web API로 통신 |
| **Dashboard** | 결과 표시 |

**제공 API** *(Swagger)* — `Blob` · `Graph`(POST/GET/PUT/DELETE `/node`) · `PII`(`/setup/setbuilding`, `/setup/setschedule`) · `Timeseries` · `Validation`

## 15. 외부 AI service

> AI 서비스가 플랫폼 **바깥**에 있고, **REST API로만 붙는다.**

- Point cloud processing
- Object detection
- **BIM**(Building Information Modeling) **geometry와의 정합성 확인**
- **as-built node(실제) 생성 → as-designed node(계획)와 연결**
- **PSI update**
- 데이터를 해석해서 **KPI 구성**

⇒ **REST API 형식에 맞는 형태로 데이터 전송**

---

# Case Study

## 16. ConSLAM dataset

- 제안한 DTC architecture와 ontology framework가 **실제 BIM, schedule, point cloud 데이터를 받아
  PII-PSI 비교와 KPI 계산까지 연결할 수 있는지 실증**
- **ConSLAM**: 런던의 Whiteley's shopping mall 건설 중 수집된 데이터

| 데이터 | 역할 | Layer |
|---|---|---|
| **IFC BIM model** | 계획된 건물 부재, geometry | Information layer / **PII** |
| **Synthetic schedule** | 계획 공정, work package | Information layer / **PII** |
| **Point cloud scans** | 실제 현장 관측 raw data | **Data layer** |
| **Progress monitoring result** | 실제 감지된 부재, 완료 시점 | Information layer / **PSI** |
| **Delay KPI** | 계획 대비 지연 지표 | **Knowledge layer** |

### 16.1 Pipeline — From Point Cloud to KPI

- 계획 데이터(BIM·schedule)와 모니터링 데이터(point cloud)가 **비교 가능한 graph entity로 변환**
- 감지된 **PSI(실제) 부재가 PII(계획)와 연결** → planned vs actual 비교로 **delay KPI 계산**
- **평가 방식**: 설계된 각 layer와 interface가 실제 데이터에서 **순서대로 작동하는지 확인**
- **Accuracy/F1 benchmark가 아니라 end-to-end 데이터 흐름의 feasibility validation** (사람이 보고 평가)

| # | 단계 | 하는 일 | 층 |
|---|---|---|---|
| 1 | **PII ingestion** | IFC + schedule → PII information graph | Information / PII |
| 2 | **Raw data registration** | Point cloud → catalog + object storage | Data |
| 3 | **Service execution** | Web API로 PII · point cloud 조회 | Web API |
| 4 | **PSI update** | As-built node ↔ as-designed 연결 | Information / PSI |
| 5 | **PII–PSI comparison** | Planned dates vs actual detection dates | Comparison |
| 6 | **Knowledge generation** | Delay KPI → time-series DB | Knowledge |

> **Delay KPI = actual detection date (PSI) vs planned end date (PII)**
> 계산된 KPI는 knowledge layer에 **시계열로 축적**되어 work package별 지연 추적에 활용

### 16.2 AI service 내부 흐름

**AI service가 받는 입력**

1. BIM에서 온 **as-designed element** — e.g. `column_415`, `slab_102`, `wall_203`
2. 각 element의 **계획 geometry** — e.g. `column_415.ply`
3. 특정 날짜의 **point cloud scan** — e.g. `scan_2022_03_29.ply`

**From Point Cloud to PSI Update** *(External AI/analysis service, Web API)*

| # | 단계 | 하는 일 |
|---|---|---|
| 1 | **Read planned elements** | DTC 플랫폼에서 as-designed BIM element와 geometry 조회 |
| 2 | **Load point cloud** | Data layer에서 scan 파일 불러오기 |
| 3 | **Align data** | point cloud를 BIM 좌표계에 정합 |
| 4 | **Detect elements** | 각 planned element를 point cloud에서 탐색 |
| 5 | **Create PSI** | as-built element와 geometry 파일 생성 |
| 6 | **Update KG** | as-built node를 as-designed node에 연결 · PII-PSI 비교 및 KPI 계산 지원 |

### 16.3 Lessons from Case Study

- Case study는 point cloud algorithm의 **정량적인 성능 평가가 아니라, feasibility 확인 목적**
- **5개 design claim이 실제 데이터 흐름에서 그대로 구현됨을 확인**

| 설계 논리 (Design claim) | Case study에서의 구현 (Evidence) |
|---|---|
| Raw data는 KG에 직접 저장하지 않음 | Point cloud는 외부 storage에 보관 — RDF에는 metadata와 file link만 기록 |
| PII와 PSI의 분리 | IFC·schedule → planned graph, 감지된 부재 → as-built graph node |
| RDF graph = semantic backbone | 부재·공정·관측·geometry 파일·KPI가 **하나의 graph로 연결** |
| AI service 출력이 digital twin을 갱신 | Detection 결과가 as-built node와 as-performed 정보 생성 |
| Knowledge는 PII–PSI 비교에서 도출 | Planned end date vs actual detection date 비교로 delay KPI 계산 |

> **Case study = ontology 기반 DTC platform의 end-to-end feasibility 검증**
> planning data → raw monitoring data → site status → plan-status 비교 → KPI knowledge

> [S05B MANUMATE](S05B-MANUMATE-제조-상호운용성.md)도 PoC였고 이 논문도 feasibility 검증이다.
> **Ch.2-2가 고른 사례 두 편 모두 "된다"의 수준이 "이 구조로 흐름이 이어진다"까지다.**
> 정량 성능은 둘 다 안 잰다. 사례 회차를 읽을 때 이 눈금을 기억해두면 과대평가를 피할 수 있다.

---

# Conclusion

## 17. Ontology as a Design Decision

- **Digital Twin Construction(DTC)를 위한 ontology와 knowledge graph framework 제안 연구**
- 단, 이번 스터디에서는 건설 도메인 자체보다,
  **도메인 지식을 고려해 어떻게 ontology & knowledge graph를 설계했는지** case study

**아래의 세 가지 주요 결정을 근거로 framework를 구성**

| # | 결정 | 내용 |
|---|---|---|
| 1 | **계획과 실제 관측의 분리** | 계획 정보(PII)와 현장에서 관측한 현재 정보(PSI)를 별도로 취급하며, **두 정보의 비교가 곧 건설 진도·지연 분석**에 사용됨 |
| 2 | **Three Layer** | raw data(point cloud)는 **Data Layer**에, 해석 결과는 **Information Layer**에, KPI 등의 파생 지식은 **Knowledge Layer**에 저장 |
| 3 | **AI service의 출력은 구조화된 KG update로** | Detection 결과를 PSI update로 graph에 반영, 이것이 다시 KPI 계산으로 연결 |

> **Ontology는 단순 용어집이 아니라, 의도(계획) · 관측(현장) · 지식(KPI)을 연결하는 설계 규칙**
> *(물론 근본적으로 형태 자체는 그냥 용어집이긴 함)*

## 다음

Ch.2-2가 여기서 끝나고, **Ch.3 평가 방법론**(S07 구조적·품질 평가 OQuaRE / S08 기능적·의미적 평가 CQ 기반)으로 넘어간다.
S06의 case study가 "사람이 보고 평가"에서 멈춘 자리를, 다음 챕터가 정면으로 다룬다.
