# S26A — CodeKGC: 코드 구조에 묶기

> Ch.7 LLM 기반 온톨로지 구축 · Day 13
> 원자료: DSBA Lab Study 강의 슬라이드 중 `01 Introduction` · `02 CodeKGC` (덱 1~20쪽, 2026-08-19)
> 참고 논문
> · Bi, Chen, Jiang, Xiong, Guo, Chen, Zhang (2023), _CodeKGC: Code Language Model for Generative
>   Knowledge Graph Construction_, ACM Transactions on Asian and Low-Resource Language Information
>   Processing
>
> 📖 [강의 목차](README.md) · [YouTube 재생목록](https://www.youtube.com/watch?v=f0WV7b3lGqM&list=PLFHGWfB_kmrs)
> 이전 [S25 멀티에이전트](S25-멀티에이전트-계획을-문서로-외부화하기.md) · 다음 [S26B Wikidata 스키마](S26B-Wikidata-스키마에-묶기.md)
> 📎 부록 [S26-1 읽는 데 필요한 것들](S26-1-읽는-데-필요한-것들.md) · [S26-2 무엇에 묶느냐가 무엇을 잘하게 될지를 정한다](S26-2-무엇에-묶느냐가-정한다.md)

**한 회차를 두 문서로 나눴다.** 이 문서가 도입부와 CodeKGC를, S26B가 Wikidata 스키마 논문과
회차 결론을 다룬다. 절 번호는 문서마다 1부터 다시 시작한다.

**이 문서는 슬라이드 내용만 담는다.** 배경 설명은 [S26-1](S26-1-읽는-데-필요한-것들.md)에,
강의 밖 해석은 [S26-2](S26-2-무엇에-묶느냐가-정한다.md)에 있다.

---

## 1. 온톨로지와 지식그래프 되짚기

| | 무엇인가 |
|---|---|
| Ontology | 데이터의 의미를 표현하는 설계도. 개념·관계·규칙을 명시적으로 정의한다 |
| Knowledge Graph | 온톨로지의 의미 구조를 바탕으로 구체적인 엔티티와 관계를 연결한 그래프 |

```
[Ontology]                        [Knowledge Graph]
Customer  places   → Order        김한결      places   → 주문 #A1024
Order     contains → Product      주문 #A1024 contains → Galaxy S25
Product   belongsTo→ Category     Galaxy S25  belongsTo→ 스마트폰
```

**온톨로지가 필요한 이유는 추론이다.** 클래스와 관계에 정의된 규칙을 이용해 명시되지 않은 새
사실을 도출한다. 온톨로지에 `Smartphone is-a Product`가 있고 지식그래프에
`Galaxy S25 종류 Smartphone`이 있으면, Galaxy S25가 Product라는 결론이 따라 나온다.

지식그래프가 데이터와 관계와 의미 구조라면 온톨로지는 의미와 추론 능력을 부여하는 층이다.

## 2. Knowledge Graph Construction의 두 갈래

비정형 텍스트에서 (head entity, relation, tail entity)를 추출해 그래프를 만드는 작업이다.

| | 방식 | 한계 |
|---|---|---|
| Pipeline KGC | NER → Entity Linking → Relation Extraction을 순차 수행 | 단계별로 오류가 누적된다. 같은 entity pair가 여러 relation을 갖는 overlapping 상황에서 누락이 발생한다 |
| Generative KGC | 관계 구조를 serialized text로 변환해 모델이 end-to-end로 단일 생성 | triple을 flat하게 나열해 topology가 훼손된다. 표준이 없다 |

Pipeline 쪽 한계의 예가 `(스티브 잡스, 창립자, 애플)`과 `(스티브 잡스, CEO, 애플)`이다. 같은
쌍에 관계가 둘인데 한 번에 하나만 잡는다.

## 3. Generative KGC의 한계 둘

| | (A) 구조 손실 | (B) 표준 부재 |
|---|---|---|
| 원인 | 관계 구조를 flat한 문자열로 나열한다 | relation·entity를 정의하는 외부 스키마 없이 자유 생성한다 |
| 결과 | 겹치는 triple, 먼 거리 관계를 제대로 표현하지 못한다 | 같은 사실이 다른 이름으로 중복된다. 기존 KB와 정합이 안 된다 |
| 예 | "워런 버핏은 네브래스카주 오마하에서 태어나 그곳에 본사를 둔 버크셔 해서웨이를 이끌고 있다" | `work_for` 대 `works for` 대 `employed_by` |

두 상황 모두 **생성을 무엇에 grounding할 것인가**의 문제다. 이 회차의 두 논문이 각각 한쪽을
맡는다. CodeKGC는 코드 구조에, Feng et al.은 Wikidata 온톨로지에 묶는다.

## 4. Why Code

**생성형 KGC의 구조적 한계** — KG의 구조화된 출력은 LLM이 학습한 자유로운 자연어와 문법·형식이
다르다. 한 문장에 여러 triple이 겹치는 경우 LLM이 정확하게 추출하기 어렵다.

**핵심 아이디어** — Code-LLM은 복잡한 추론과 구조 예측에서 강점을 보인다. Code-LLM이 자연어보다
구조를 더 잘 모델링한다는 전제에서, 자연어 입력을 코드 형식으로 re-structure해 Code-LLM에 넣는다.

장치가 둘이다.

1. **Schema-aware code prompt** — 스키마를 class 상속으로 인코딩해 구조 인식력을 높인다
2. **Rationale-enhanced generation (optional)** — 추론 중간 단계(CoT)를 삽입해 추출 능력을 높인다

## 5. Overview — KGC를 코드 생성으로 재정의

입력·출력의 형식을 바꾼다.

```
Training Instance (T, G)      T: 입력 텍스트, G: 생성할 KG (triple의 집합)
자연어 형식 (T, G)  →  코드 형식 (T_c, G_c)로 재구성
Code-LLM이 G_c를 생성  →  이후 다시 G로 역변환
```

프롬프트 구성은 넷이다.

1. 스키마 class 정의
2. In-context Prompts + rationale(optional) + 정답 triple 제공
3. Task Prompt
4. 최종 출력은 rationale(optional) + triple

## 6. 장치 ① — Schema-aware Prompt Construction

KG의 스키마를 Python class로 인코딩한다. 스키마는 KG를 구성하는 entity·relation과 그 제약이
표현된 것이고, 이를 Python class와 상속 구조로 표현한다.

```python
# Base Definition
class Rel:
    def __init__(self, name: str):
        self.name = name
class Entity:
    def __init__(self, name: str):
        self.name = name

# Schema Information
class Work_for(Rel):
    def __init__(self, name: str):
        self.name = name
class person(Entity):
    def __init__(self, name: str):
        super().__init__(name=name)

class Triple:
    def __init__(self, head, relation, tail):
        self.head = head
        self.relation = relation
        self.tail = tail
class Extract:
    def __init__(self, triples):
        self.triples = triples
```

- **Base Definition** — `class Entity`, `class Rel`이 모든 entity와 relation의 최상위 기본 class다
- **Schema Information** — `class person(Entity)`, `class Work_for(Rel)`처럼 상속으로 type을
  표현한다. 데이터셋별로 `organization(Entity)`, `Live_in(Rel)` 등이 다양하게 붙는다
- `class Triple(head, relation, tail)`이 하나의 triple을 구성하고, `class Extract(triples)`가
  triple의 집합, 곧 최종 추출 결과다

## 7. 장치 ② — Rationale-enhanced Generation

최종 triple을 바로 생성하지 않고 중간 추론 단계를 거치게 한다. CoT를 KGC에 적용한 것이고
선택 모듈이다. 복잡한 추출 과정을 단순한 하위 단계로 분해해 정확도를 높인다.

```
Step 1  Relation 식별 — 문장에서 relation 후보를 추출
Step 2  Entity 추출 — 해당 relation에 연결된 entity 추출
Step 3  Triple 생성 — 앞선 결과를 조합해 최종 triple 생성
```

일반 생성과 CoT 생성의 확률 분해가 이렇게 갈린다.

```
일반    p(G_c | T_c, P) = ∏_i p_LM(g_i | T_c, P, g_<i)

CoT     p(G_c | T_c, P) = p(G_c | T_c, P, R) · p(R | P, T_c)
        p(G_c | T_c, P, R) = ∏_i p_LM(g_i | T_c, P, R, g_<i)
        p(R | P, T_c) = ∏_i p_LM(r_i | P, T_c, r_<i)
```

프롬프트에는 후보 relation과 후보 entity를 주석으로 적어 넣는다.

```python
# The candidate relations in this sentences
# Rel('Work for')
# The candidate entities in this sentences
# organization('Washington Institute for Near East Policy')

extract = Extract([
  Triple(person('Marvin Feuerwerger'), Rel('Work for'),
         organization('Washington Institute for Near East Policy')),
])
```

## 8. 실험 설정

| 데이터셋 | 도메인 | Entity 타입 | Relation 타입 |
|---|---|---|---|
| ADE | 의학 | 2 (disease, drug) | 1 (adverse effect) |
| CoNLL04 | 뉴스 | 3 (person, org, location) | 4 (work for, live in 등) |
| SciERC | 과학 논문 | 6 (method, task 등) | 7 (hyponym-of, used-for 등) |

- **Backbone** — text-davinci-002 / 003 (RQ1~3), T5 / Code T5 (RQ4), code-davinci-002 (ablation)
- **Baseline** — UIE(Unified Information Extraction, 여러 정보 추출을 하나의 통합 생성
  framework로 처리하는 supervised 모델), Vanilla Prompt
- **평가 지표** — Relation strict F1 (head, relation, tail이 모두 일치해야 정답). entity 타입
  채점은 제외하고 3회 실행 후 평균을 낸다

## 9. RQ1 — baseline보다 좋은가

이 논문은 1-shot 설정을 zero-shot이라 부른다. 다만 모델에게 relation을 동의어로 보여준다
(맞혀야 할 relation이 `work_for`이면 in-context에서는 `employed_for`로 보여준다).

| | Comparable SOTA | ADE | CoNLL04 | SciERC |
|---|---|---:|---:|---:|
| Zero-Shot | UIE | 24.3 | 16.1 | 10.3 |
| | Vanilla Prompt (text-davinci-002) | 41.2 | 18.4 | 12.2 |
| | Vanilla Prompt (text-davinci-003) | 41.7 | 30.5 | 18.1 |
| | CodeKGC (text-davinci-002) | 42.5 | 35.8 | 15.0 |
| | **CodeKGC (text-davinci-003)** | **43.7** | **41.6** | **19.5** |
| Few-Shot | UIE | 50.3 | 39.0 | 19.2 |
| | Vanilla Prompt (text-davinci-002) | 45.7 | 28.2 | 14.1 |
| | Vanilla Prompt (text-davinci-003) | 58.8 | 43.2 | 18.8 |
| | CodeKGC (text-davinci-002) | 61.5 | 42.7 | 18.5 |
| | **CodeKGC (text-davinci-003)** | **64.2** | **49.6** | **24.7** |

- 모든 데이터셋에서 baseline 대비 향상했고, few-shot(3-shot)에서도 대부분 향상한다
- 여러 relation과 entity로 구성된 CoNLL04에서 향상 폭이 크다 (+11.1)
- ADE(의학 용어)에서 zero-shot 성능 향상이 제한적이다 (+2.0)

## 10. RQ2 — 두 장치가 각각 기여하는가

CoNLL04에서 두 모듈을 제거하고 성능을 비교했다.

| Backbone | CodeKGC | w/o Schema | w/o Rationale |
|---|---:|---:|---:|
| text-davinci-002 | 43.2 | 40.0 | 41.1 |
| text-davinci-003 | 49.6 | 44.8 | 46.5 |

두 모듈 모두 성능 향상에 기여한다. **스키마 모듈의 기여가 더 크다.** 다만 rationale은 샘플 수가
적을 때 오히려 역효과를 내는 경우가 있다.

## 11. RQ3 — 겹치는 triple을 더 잘 처리하는가

ADE에서 서로 겹치는 triple만 추출해 few-shot 평가를 진행했다. 비교 대상은 CodeKGC와 Vanilla
Prompt이고 backbone은 text-davinci-002와 code-davinci-002다.

| Backbone | Vanilla Prompt | CodeKGC |
|---|---:|---:|
| text-davinci-002 | **51.8** | 48.1 |
| code-davinci-002 | 53.6 | **59.7** |

- code-davinci-002에서는 CodeKGC가 Vanilla를 능가한다. 중첩 triple 처리에 우수하다
- text-davinci-002에서는 오히려 Vanilla가 높다. **코드 특화 backbone이 있어야 이점이 발휘된다**

정성 예시로는 `Prenatal cytomegalovirus (CMV) infection associated with severe brain damage...`
문장에서 CodeKGC가 `brain damage`와 `prednisolone`·`azathioprine`을 함께 잡는 반면 Vanilla는
`cytomegalovirus`와 `brain damage`만 잡아 겹치는 쪽을 놓친다.

## 12. RQ4 — 소형 code LM으로도 가능한가

GPT-3.5 대신 소형 모델(T5, CodeT5 770M)을 직접 pre-train했다. 자체 구축한 code-format corpus를
활용한다.

| Data Source | Sentences | Triples | Relations |
|---|---:|---:|---:|
| T-REx | 838,821 | 4,378,318 | 10,496 |
| TEKGEN | 6,310,061 | 10,609,697 | 1,013 |
| KELM | 15,616,551 | 35,117,608 | 59,904 |
| Overall | 22,765,433 | 50,105,623 | 71,413 |

ADE에서 fine-tune 후 성능을 비교했다.

| | T5 | CodeT5 |
|---|---:|---:|
| Restructured Code Pre-trained | **75.6** | 68.6 |
| Not Pre-trained | 52.2 | 58.8 |

UIE가 81.1이다. 슬라이드가 물음표를 붙인 지점은 **pre-training을 하면 T5가 CodeT5보다 높다**는
역전이다.

해석은 이렇다. KGC를 코드 생성으로 풀 때는 텍스트 의미 이해와 코드 생성 능력이 모두 필요한데,
제한된 파라미터로는 둘을 겸비하기 어렵다. 이는 대형 모델에서만 발현된다. RQ3에서 강력한 code
backbone이 필요하다고 했고, RQ4에서 소형 모델로는 힘든 작업임이 확인된다.

## 13. 정리

**기여** — KGC를 코드 생성으로 재정의해 코드 구조로 KG의 구조를 보존한다. Schema-aware code
prompt가 스키마를 class 상속으로 인코딩하고, Rationale-enhanced generation이 relation → entity
→ triple의 3단계 추론을 넣는다.

**논의**

- 구조 예측 문제에서는 데이터의 형식이 중요하다 (flat text, constrained decoding << code)
- 코드가 전용 구조 언어(UIE)보다 범용적이다
- 코드는 자연어 요소와 구조 표현을 겸비해 구조적 추론에 유리하다
- 다만 Code-LLM의 성능에 좌우된다. 소형 모델에서 격차가 크다

---

## 관련 문서

- 이어지는 본편 — [S26B Wikidata 스키마에 묶기](S26B-Wikidata-스키마에-묶기.md)
- 부록 — [S26-1 읽는 데 필요한 것들](S26-1-읽는-데-필요한-것들.md) ·
  [S26-2 무엇에 묶느냐가 정한다](S26-2-무엇에-묶느냐가-정한다.md)
