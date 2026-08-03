# S03A — 텍스트 기반 온톨로지 자동 추출 ①: Terms Layer (P-Mod)

> 2026-07-30 | 온톨로지, OntologyLearning, LayerCake, TermExtraction, P-Mod, UMLS, Medline

> 📖 [강의 목차/진행](README.md) · 이전: [S02 설계 원칙](S02-설계-원칙.md) · 다음: [S03B 비분류 관계 학습](S03B-비분류-관계-학습.md)
>
> 📎 부록: [S03-1 두 논문에서 읽히는 것](S03-1-자동구축의-천장.md) · [S03-2 추출 이후 남는 것들](S03-2-추출-이후-빠진-것들.md)

## 한 줄 요약

텍스트에서 온톨로지를 뽑는 일은 **Layer Cake**(Terms → Synonyms → Concepts → Taxonomy → Relations → Rules)로
아래에서 위로 쌓인다. 이 회차의 두 논문은 각각 맨 아래(Terms)와 위쪽(Relations)을 담당하는데,
**둘 다 같은 원리를 쓴다 — 언어적 기법으로 후보를 넓게 모으고(recall), 통계적 기법으로 좁힌다(precision).**
S03A는 Terms 층의 논문으로, 핵심 아이디어는 **"진짜 용어는 단어를 바꾸기 어렵다"**(P-Mod)다.

---

## 1. 온톨로지 — 세 구성 요소로 다시 정의

**정의** — 형식적(formal) · 명시적(explicit)으로 규정된, **공유된 개념화**(shared conceptualization)

> S02 4.2절의 Gruber 정의가 그대로 다시 나온다.

**구성 요소**

| 기호 | 이름 | 정의 |
|---|---|---|
| **C** | concept | 도메인의 핵심 용어 혹은 class |
| **H** | 분류 관계 | 개념 간 상하위 (IS-A) · `H ⊆ C × C` |
| **R\*** | 비분류 관계 | IS-A 외의 의미 관계 · `R* ⊆ C × C × ` **`string`** |

| | 분류 관계 (Taxonomic) | 비분류 관계 (Non-taxonomic) |
|---|---|---|
| 관계 종류 | **IS-A 하나로 고정** | **무한히 많음** (인과 · 목적 · 속성 등) |
| 정해야 할 것 | 개념쌍 | 개념쌍 + **관계 이름** |
| 구조 | 트리 구조 (계층적) | 그물망 |
| 예시 | 폐암 IS-A 암 | 인과: 흡연 → 암 |

> **`R*`의 정의에 `string`이 들어간 게 이 회차 전체의 열쇠다.** 관계에 **이름을 붙여야** 하고,
> 그 이름을 어디서 가져오느냐가 [S03B](S03B-비분류-관계-학습.md) 논문의 문제다.
> 분류 관계는 이름이 IS-A로 고정이라 개념쌍만 찾으면 되지만, 비분류 관계는 **찾기 + 이름 붙이기** 두 일이다.

## 2. 수동 구축의 한계 — 왜 자동화인가

| 한계 | 문제점 | 결과 |
|---|---|---|
| **Scale** | 도메인 하나에도 얽힌 개념·관계가 수천~수만 개 | 사람이 수동으로 열거하는 것이 **불가능** |
| **Cost** | 전문가·공학자의 노동 집약적 작업 필요 | **완성률이 높은 프로젝트가 드묾** |
| **Dynamicity** | 새 용어·지식이 계속 유입됨 | **Knowledge가 뒤처지는** 문제 발생 |

→ **텍스트에서 자동으로 학습(Ontology Learning)하는 방법이 필요함**

## 3. Ontology Learning Layer Cake

- 텍스트에서 온톨로지를 뽑는 과정은 **여러 layer가 아래에서 위로 쌓이는 구조**
- **각 layer는 아래의 결과를 입력으로 받음**

| Layer | 하는 일 | 예시 | |
|---|---|---|---|
| **Rules** | 논리적 제약 · 추론 규칙 | 흡연은 폐암의 위험 요인 → 흡연자는 폐암 위험군 | |
| **Relations** | 개념 간 비분류 관계 + 라벨 | 흡연이 폐암을 유발함 | ← **2nd Paper** |
| **Taxonomy** | 개념 간 IS-A 계층 | 폐암 IS-A 암 | |
| **Concepts** | 하나의 클래스로 묶어 정의함 | Class [폐암] | |
| **Synonyms** | 같은 뜻의 표현 묶기 | 폐암 = lung cancer = 폐 종양 | |
| **Terms** | 텍스트에서 용어 후보 추출 | 문서에서 "폐암" 추출 | ← **1st Paper** |

```
        ┌──────────────┐
        │    Rules     │
      ┌─┴──────────────┴─┐
      │    Relations     │ ← 2nd Paper
    ┌─┴──────────────────┴─┐
    │ Concept Hierarchies  │
  ┌─┴──────────────────────┴─┐
  │        Concepts          │
┌─┴──────────────────────────┴─┐
│         Synonyms             │
├──────────────────────────────┤
│          Terms               │ ← 1st Paper
└──────────────────────────────┘
```

## 4. Today's Paper — 전통적인 Ontology 구축 파이프라인

두 논문이 공유하는 2단 구조다.

```
① 언어적 후보 추출   품사 · 구문 분석을 바탕으로 후보 선정   → recall 확보
② 통계적 선별        통계 지표를 바탕으로 최종 선택          → precision 확보
```

| | **Paper 1** (Terms Layer) | **Paper 2** (Relations Layer) |
|---|---|---|
| 푸는 문제 | 사전에 없는 **새 용어 발견** | 개념 간 **비분류 관계 발견 → 라벨링** |
| ① 언어적 후보 | 명사구(NP) 추출 | 동사구(VP) 패턴 |
| ② 통계적 방법 | **P-Mod** | Web-scale statistics (co-occurrence) |
| 데이터 출처 | **고정된 코퍼스** | **웹** (검색엔진) |

---

# Paper 1 — Finding New Terminology in Very Large Corpora

## 5. Term Extraction의 필요성

**Motivation**

- Medical · Biological 분야의 문헌이 지속적으로 발간됨 → **domain-specific terminology를 자동 추출·관리할 필요** 증가
- 수작업 식별은 cost-time 소요가 높음 → 자동화 필요

**Challenge**

- **도메인 용어의 85% 이상이 multi-word unit** → 단일어보다 추출이 어려움
  - ex. [생물학] `long terminal repeat`

**Research Gap**

- (당시의) 표준적인 방법: **linguistic filtering**(POS tagging, phrase chunking) + **statistical measures**(C-value, t-test)
- 그러나 이러한 방법은 용어의 **linguistic property를 거의 활용하지 못함** ← **Frequency 기반 통계의 한계**

| Term | 용어인가? | Frequency |
|---|---|---|
| `long terminal repeat` | **용어** | 434회 (낮음) → **비용어라 오판** |
| `t cell response` | **비용어** | 2410회 (높음) → **용어라 오판** |

이 표 하나가 논문 전체의 출발점이다. 빈도는 용어성과 상관이 없을 수 있다.

## 6. Methods ① — Term Candidate 추출 (언어적 단계)

**Corpus 구축**

- Medline 초록 약 **513,000개** → **104M word** 생의학 코퍼스
  - Medline: 생의학 분야의 논문 초록 DB
  - MeSH query의 다음 분류 tag만 추출: `transcription factors`, `blood cells`, `human`

**Noun Phrase 후보 추출 (linguistic filtering)**

- **GENIA POS tagger**(생의학 text에 특화)로 품사 태깅 → **YamCha chunker**(SVM 기반)로 Noun Phrase 인식
- 근거: **전문 용어의 대다수가 noun phrase 안에 존재**
- 길이: bigram / trigram / quadgram NP
- stop word, `and`·`or` 포함 NP 제외

**정규화 & Cut-off**

- **Morphological normalization**: NP의 head 명사를 정규화 (ex. cell / cells → cell)
- **Frequency cut-off**: 저빈도 노이즈 제거 (bigram ≤ 10, trigram ≤ 8, quadgram ≤ 6)

## 7. Methods ② — Paradigmatic Modifiability (P-Mod)

### 7.1 Main Idea — 진짜 용어는 '단어를 바꾸기 어렵다'

**P-Mod** = 용어의 한 slot을 다른 단어로 **대체하기 어려운 정도**

- **용어**: 대체가 어려움 (limited P-Mod)
- **비용어**: 대체가 자유로움

```
long terminal repeat   →  'repeat' 자리에 다른 단어를 넣기 어려움
                       →  P-Mod가 높음  →  용어일 가능성이 높음

t cell response        →  t cell activation, t cell count, …
                       →  P-Mod가 낮음  →  용어일 가능성이 낮음
```

> **읽을 때 헷갈리는 지점** — 이름은 "Modifiability(수정 가능성)"인데 **값이 높을수록 수정하기 어렵다**는 뜻이다.
> 아래 `mod_sel` 식을 보면 이유가 보인다 — 슬롯이 고정돼 있으면 분모 `f(sel)`이 분자에 가까워져 값이 1에 근접하고,
> 자유롭게 대체되면 `f(sel)`이 훨씬 커져 값이 0에 가까워진다. **지표가 재는 건 사실 '비(非)수정 가능성'이다.**

### 7.2 K-modifiability 계산

| 단계 | 내용 | 식 |
|---|---|---|
| 1 | **Slot selection** — 몇 개의 자리를 비울까 | `nCk = n! / (k!(n-k)!)` |
| 2 | **Selection의 modifiability** — 그 자리가 얼마나 고정되었는가 | `mod_sel = f(n-gram) / f(sel)` |
| 3 | **k-modifiability** — 같은 k의 모든 조합을 곱함 | `mod_k(n-gram) = ∏_{i=1}^{s} f(n-gram) / f(sel_i, n-gram)` |
| 4 | **P-Mod** — 모든 k-mod를 곱함 | `P-Mod(n-gram) = ∏_{k=1}^{n} mod_k` |

`long terminal repeat`의 k=1 (한 자리 비우기):

```
long terminal ____        ← 이 자리에 'repeat'이 오는 비율
long ______ repeat
____ terminal repeat
                ↓
      각 비율을 모두 곱함 → mod₁
                ↓
P-Mod(3-gram) = mod₁ × mod₂ × mod₃
```

### 7.3 실제 결과

| n-gram | freq | mod₁ | mod₂ | P-Mod (k=1,2) |
|---|---|---|---|---|
| `long terminal repeat` (용어) | 434 | **0.91** | 0.03 | **0.03** |
| `t cell response` (비용어) | 2,410 | 0.06 | 0.00008 | **0.00005** |

`long terminal repeat`의 k=1 내역: `k₁ terminal repeat` 460회→0.94 · `long k₂ repeat` 448회→0.97 ·
`long terminal k₃` 436회→0.995 → mod₁ = 0.91. **세 자리 모두 거의 고정돼 있다.**

**순위 비교 — 이게 논문의 결정적 그림이다.**

| trigram | 빈도 | t-test | **P-Mod** |
|---|---|---|---|
| `long terminal repeat` (용어) | 434 | 242위 | **24위** |
| `t cell response` (비용어) | 2,410 | 24위 | **1249위** |

빈도 기반 지표가 정확히 뒤집어 놓은 두 항목을 P-Mod가 바로잡는다.

**한계** — 일부 용어 처리가 어려운 phrase가 존재한다.

- `bone marrow cell`: `cell` 자리에 올 수 있는 다른 단어가 많음 → **실제 용어이지만 P-Mod 순위가 떨어짐**

## 8. Methods of Evaluation

**Gold Standard: 무엇을 정답 용어로 볼 것인가?**

| | 방식 |
|---|---|
| 기존 방식 | domain expert가 **상위 후보를 직접 판정** |
| **본 논문** | **UMLS Metathesaurus**에 등재된 용어를 정답으로 간주 → **객관적인 검증 가능** |

> UMLS Metathesaurus = 미국 국립 통합 의학 용어 체계. [S01 7.2절](S01-기본개념과-실사례.md)에서
> "무엇이 동일한 개념인가"를 정리하는 층으로 배운 그것이다. **여기서는 정답지 역할을 한다.**

**Baselines**

| | 내용 |
|---|---|
| **P-Mod** (ours) | |
| t-test | |
| **C-value** | 빈도 + **중첩 구조 보정** |

- C-value가 보정하는 것: `soft contacts lens`가 용어면 그 안의 `contact lens`도 용어다
  → **'독립적인 용어인가, 혹은 긴 용어의 일부분인가'**를 고려하여 빈도를 계산

**평가 지표** — Precision, Recall. 각 지표로 후보를 순위화 → **상위 m% 구간마다** precision/recall 측정

## 9. Results

### 9.1 Main Results

- Precision: 뽑은 것 중 **진짜 용어의 비율** / Recall: 전체 중 **진짜 용어를 뽑은 비율**

**Precision** (상위 % 구간별)

| | 구간 | P-Mod | t-test | C-value |
|---|---|---|---|---|
| **Bigrams** | 1% | **0.82** | 0.62 | 0.62 |
| | 10% | **0.53** | 0.42 | 0.41 |
| | 20% | **0.42** | 0.35 | 0.34 |
| | 30% | **0.37** | 0.32 | 0.31 |
| | baseline | 0.22 | 0.22 | 0.22 |
| **Trigrams** | 1% | **0.62** | 0.55 | 0.54 |
| | 10% | **0.37** | 0.29 | 0.28 |
| | 20% | **0.29** | 0.23 | 0.23 |
| | 30% | **0.24** | 0.20 | 0.19 |
| | baseline | 0.12 | 0.12 | 0.12 |
| **Quadgrams** | 1% | 0.43 | **0.50** | **0.50** |
| | 10% | **0.26** | 0.24 | 0.23 |
| | 20% | **0.20** | 0.16 | 0.16 |
| | 30% | **0.18** | 0.14 | 0.14 |
| | baseline | 0.08 | 0.08 | 0.08 |

**Recall** — 해당 recall 수치를 달성하기 위해 살펴본 **상위 단어 %** (낮을수록 좋음)

| | recall | P-Mod | t-test | C-value |
|---|---|---|---|---|
| **Bigrams** | 0.5 | **29%** | 35% | 37% |
| | 0.7 | **51%** | 56% | 59% |
| | 0.9 | **82%** | 83% | 85% |
| **Trigrams** | 0.5 | **19%** | 28% | 30% |
| | 0.7 | **36%** | 50% | 53% |
| | 0.9 | **68%** | 77% | 84% |
| **Quadgrams** | 0.5 | **20%** | 28% | 30% |
| | 0.7 | **34%** | 49% | 53% |
| | 0.9 | **61%** | 79% | 82% |

> **"모든 지표에서 우위"는 정확히는 한 칸 예외가 있다.** Quadgram 상위 1% 구간에서 P-Mod 0.43 <
> t-test·C-value 0.50이다. 나머지 전 구간은 P-Mod가 앞선다. 가장 긴 n-gram의 최상위 구간이라
> 표본이 적은 지점이기도 하다.

### 9.2 통계적 유의성 — McNemar test

- **두 방법이 같은 대상에 대해 내린 판정이 유의미하게 다른지** 검정하는 방법
- **두 방법의 판정이 갈리는 경우**를 주목한다
  - P-Mod는 맞고 t-test는 틀림 → P-Mod가 더 잘함
  - P-Mod는 틀리고 t-test는 맞음 → t-test가 더 잘함
- 상위 1%, 2%, …, 10% 지점의 통계적 유의성 검증 (P-Mod가 precision이 높은지 확인)

측정 지점(measure points) 10~100에 대해 bigram/trigram/quadgram별로 t-test·C-value 대비
유의한 차이의 개수를 센다. 지점 수가 늘수록 유의한 차이의 개수도 거의 비례해 늘어난다
(bigram 100지점에서 t-test 대비 93 / C-value 대비 100).

### 9.3 Domain Independence and Corpus Size

**P-Mod의 우수성은 코퍼스의 압도적 크기 때문인가?**

- 코퍼스를 **1/10로 축소**(100M → 10M)하여 trigram 대상 실험 수행
- **Precision/Recall 모두 성능 능가**, McNemar test도 유의미한 결과

→ **Domain Independence: 도메인 특화 요소가 없음!**

## 10. Conclusion & Limitations

**Contribution**

- 단순 빈도가 아닌, **언어적 성질에 기반한 P-Mod 지표를 정식화**
- 표준 지표(t-test, C-value) 대비 precision/recall에서 강점을 보임
- **Gold standard 기반 전체 후보를 객관적으로 검증**

**Limitations**

- **범용적인 단어의 경우, 용어 포착에 어려움이 존재함**
- Domain Independence에 대한 언급이 있지만, **실제 실험은 진행하지 않음**
- **Multi-word(n ≥ 2) 전용 방법론** → **단일어에는 적용이 불가능함**

> **9.3절과 이 Limitation은 모순이 아니다.** 9.3은 *방법론에 도메인 특화 요소가 없다*는
> **설계상의 주장**이고, Limitation은 *다른 도메인에서 실측하지 않았다*는 **실증의 부재**다.
> 같은 단어(Domain Independence)로 두 가지를 말하고 있어서 헷갈린다.
> 실제로 축소 실험은 **코퍼스 크기**를 바꿨을 뿐 도메인은 그대로 생의학이다.

## 다음

[S03B](S03B-비분류-관계-학습.md)는 Layer Cake의 위쪽 — **Relations 층**으로 올라간다.
같은 2단 구조(언어적 후보 → 통계적 선별)를 쓰지만, 명사구 대신 **동사구**를 쓰고
고정된 코퍼스 대신 **웹**을 쓴다. 그리고 1절에서 짚은 `string` 라벨 문제를 정면으로 다룬다.
