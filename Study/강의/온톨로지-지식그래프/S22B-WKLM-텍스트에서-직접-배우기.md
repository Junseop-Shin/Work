# S22B — WKLM: 텍스트에서 직접 배우기

> Ch.6 언어모델+KG · Day 11
> 원자료: DSBA Lab Study 강의 슬라이드 중 `03 WKLM` · `04 Summary` (덱 31~48쪽)
> 참고 논문
> · Xiong, Du, Wang, Stoyanov (2020), _Pretrained Encyclopedia: Weakly Supervised
>   Knowledge-Pretrained Language Model_, ICLR 2020
>   ([arXiv:1912.09637](https://arxiv.org/abs/1912.09637))
>
> 📖 [강의 목차](README.md) · [YouTube 재생목록](https://www.youtube.com/watch?v=f0WV7b3lGqM&list=PLFHGWfB_kmrs)
> 이전 [S22A COMET](S22A-COMET-언어모델로-KG를-만든다.md) · 다음 S23 LLMs4OL
> 📎 부록 [S22-1 읽는 데 필요한 것들](S22-1-읽는-데-필요한-것들.md) · [S22-2 방향이 뒤집힌 회차](S22-2-방향이-뒤집힌-회차.md)

**한 회차를 두 문서로 나눴다.** [S22A](S22A-COMET-언어모델로-KG를-만든다.md)가 도입부와 COMET을,
이 문서가 WKLM과 회차 결론을 다룬다. 절 번호는 문서마다 1부터 다시 시작한다.

**이 문서는 슬라이드 내용만 담는다.** 배경 설명은 [S22-1](S22-1-읽는-데-필요한-것들.md)에,
강의 밖 해석은 [S22-2](S22-2-방향이-뒤집힌-회차.md)에 있다.

예문은 `Spider-Man is a fictional superhero created by writer-editor Stan Lee`로 이어진다.

---

## 1. Overview — Weakly Supervised Knowledge-Pretrained LM

출발 질문은 사전학습 LM이 실세계 지식을 정말 갖고 있는지, 얼마나 갖고 있는지다.

| 기여 | 내용 |
|---|---|
| 평가 확장 | 기존의 단일 토큰 제약을 넘어 멀티토큰 엔티티를 다루는 엔티티 랭킹 설정을 제안한다 |
| 약한 지도 목적함수 | 엔티티 치환 판별. 텍스트에서 직접 엔티티 지식을 학습한다 |
| SOTA | 엔티티 관련 QA 3개 데이터셋과 FIGER 엔티티 타이핑 |

대표 수치는 zero-shot fact completion Hits@10이다. BERT-base 11.3, BERT-large 16.1, GPT-2 16.3,
WKLM 28.9다. 엔티티 관련 QA 4개 데이터셋 평균 +2.7 F1, FIGER 정확도 +5.7이다.

## 2. 문제 제기 — LM의 Knowledge

**긍정적 정황**

- WNLI · ReCoRD · SWAG의 SOTA가 사전학습 모델에서 나온다. 이 과제들은 텍스트 입력만으로 정확한
  예측에 필요한 정보가 부족하도록 설계되어 외부 지식이 필요하다
- Logan et al. (2019), Petroni et al. (2019) — 단일 토큰 엔티티 fact completion에서 랜덤보다
  훨씬 낫고 전용 관계 추출 모델과 비등하다

**그런데** 기존 사전학습 목적함수는 토큰 레벨에서 정의되며 엔티티 중심 지식을 명시적으로
모델링하지 않는다. 저자들의 Wikidata 10개 관계 실험에서 기존 사전학습 모델은 엔티티 레벨 지식을
제한적으로만 인코딩하고 있었다.

**기존 대안의 문제 (S19~S21)** — ERNIE(Zhang 2019)와 KnowBERT(Peters 2019)는 외부 KB를 써서
엔티티 지식을 넣는다. 그 대가로 추가 데이터 처리, 추가 메모리, 모델 구조 변경이 따른다.
WKLM의 기여는 비정형 텍스트에서 직접, 구조 변경 없이 해내는 것이다.

## 3. Entity Replacement

원문을 참인 지식 진술로 두고, 엔티티를 같은 타입의 다른 엔티티로 바꾼 문서를 거짓으로 둔다.

| 단계 | 내용 |
|---|---|
| 1 | 엔티티 멘션 인식 → 위키피디아 엔티티 링킹 |
| 2 | 원문 = 참인 지식 진술 (positive) |
| 3 | 같은 타입의 다른 엔티티로 치환 (negative) |
| 4 | "치환되었는가?" 이진 판별 |

```
Original:  ... created by writer-editor Stan Lee and writer-artist Steve Ditko.
           He first appeared in the anthology comic book American comic books
           published by Marvel Comics.
Replaced:  ... created by writer-editor Bryan Johnson and writer-artist Steve Ditko.
           ... published by DC Comics.
```

타입 제약 조회 경로는 이렇게 흐른다.

```mermaid
flowchart LR
    M["Marvel Comics"] -->|entity linking| Q["Q173496"]
    Q -->|"P31 (instance of)"| T["Q1320047<br/>book publishing company"]
    T --> P["같은 타입 풀<br/>DC Comics · Dark Horse · Image Comics ..."]
    P -->|random sample| R["DC Comics"]
```

목적함수는 치환 여부에 대한 이진 로그우도다.

```
J = 1[e∈E⁺] · log P(e|C)  +  (1 − 1[e∈E⁺]) · log(1 − P(e|C))
```

## 4. 타입 제약

| | Ch.5 · S13 TransE (Bordes et al. 2013) | Ch.6 · S22 WKLM (Xiong et al. 2020) |
|---|---|---|
| 조작 | 트리플 (h, r, t)의 subject 또는 object를 같은 타입의 랜덤 엔티티로 바꿔 negative triple 생성 | 비정형 텍스트를 사실 진술로 취급하고 동일한 조작을 가해 negative context 생성 |

**타입을 강제하는 이유**

| | 문장 자연스러움 | 모델이 쓰는 단서 | 결과 |
|---|---|---|---|
| 다른 타입으로 치환 | 깨진다 | 언어적 단서 | 지식 학습 실패 |
| 같은 타입으로 치환 | 유지된다 | 사실 지식 | 의도한 학습 |

MLM은 토큰 레벨이라 negative 신호가 약하고, Entity Replacement는 엔티티 레벨이라 강하다.

## 5. 데이터 준비

English Wikipedia에서 이렇게 만든다.

| 단계 | 내용 |
|---|---|
| ① | 앵커 링크로 주석된 엔티티를 먼저 확보한다 |
| ② | 그 엔티티들의 Wikidata alias를 문자열 매칭해 나머지 멘션도 포착한다 |
| ③ | 각 문서를 512-token 청크로 분할한다 |
| ④ | 청크당 10회 복제한다. 치환 위치마다 다른 negative 엔티티를 넣는다 |

각 Wikipedia 엔티티(title)는 Wikidata의 고유 노드에 대응한다.

엔티티 링크는 사전학습에만 필요하고 fine-tuning 시에는 불필요하므로 다운스트림 오버헤드가 0이다.
off-the-shelf 엔티티 링킹 도구로 더 큰 코퍼스에 확장할 수 있다.

## 6. 치환 전략

| 장치 | 내용 |
|---|---|
| ① 타입 조회 | Wikidata instance of (P31)를 쓴다. 올바른 타입이 여러 개면 무작위로 하나 선택한다 |
| ② 인접 치환 금지 | 치환된 두 엔티티 사이에 최소 하나의 미치환 엔티티가 있어야 한다 |
| ③ 표기 무작위 샘플링 | 치환 시 해당 엔티티의 alias 집합에서 무작위 문자열을 고른다 |

②를 왜 두는가. 한 문장의 모든 엔티티를 치환하면 결과 문장이 우연히 올바른 조합을 만들어낼 수
있다. 치환(●)과 미치환(○)이 `● — ○ — ●`로 번갈아야 하고 `● — ●`는 금지된다.

이 세 디테일이 "지식 없이도 풀 수 있는 지름길"을 막는 장치다. 데이터 설계가 목적함수만큼 중요한
사례다.

## 7. 모델 구조 & 예측 방식

```
...  created by writer-editor [ Stan Lee ] and writer-artist ...
```

엔티티 스팬의 앞뒤 경계 표현 `h_before`와 `h_after`를 이어붙이고 linear + dropout 0.05를 거쳐
`P(e | C)`로 진짜인지 가짜인지를 낸다.

- 백본은 BERT-base와 동일하다. Transformer 12 layers, hidden 768
- 초기화는 자체 BERT 재구현이다. Fairseq MLM 구현, BooksCorpus + English Wikipedia, 2M updates로
  원 BERT(1M)보다 오래 학습됐다

엔티티 토큰 자체를 쓰지 않고 주변 문맥의 표현으로 판단한다. 표면 정보 의존을 차단한다.

## 8. MLM 병용 & 학습 세팅

WKLM은 엔티티 스팬에 Entity Replacement를, 그 바깥에 Masked LM을 함께 쓴다.

**MLM 쪽 두 가지 수정**

- 마스크를 엔티티 스팬 바깥으로 제한한다
- 마스킹 비율을 15%에서 5%로 낮춘다. 문맥을 많이 가리면 지식 학습 단서가 손상된다

| 학습 설정 | 값 |
|---|---|
| updates | 약 1M |
| batch size | 128 |
| optimizer | Adam |
| learning rate | 1e-5 |
| weight decay | 0.01 |
| 자원 | 32 × V100, 3일 |

다운스트림 fine-tuning 시 추가 데이터 처리도, 추가 메모리도, BERT 구조 변경도 필요 없다.
그냥 BERT처럼 쓰면 된다. ERNIE·KnowBERT 대비 결정적 차별점이다.

## 9. Zero-shot Fact Completion 설계

| 단계 | 예 |
|---|---|
| ① Wikidata triple | { Paris, CapitalOf, France } |
| ② 자연어 문장 (수작업 템플릿) | "the capital of France is Paris" |
| ③ cloze 질의 (목적어 제거) | "the capital of France is ___" |
| ④ 엔티티 랭킹 (후보 집합) | Hits@10 |

- 10개 흔한 관계이고 관계당 1,000개 cloze 예제다
- negative 후보는 해당 관계의 모든 목적어 엔티티다. 대체로 같은 타입이라 구별이 더 어렵다

전통적 KB completion은 학습 트리플에 접근할 수 있다. 여기서는 zero-shot으로, 모델이 자연어에서
관계 지식을 자동 도출했는지 확인한다.

**모델별 점수 계산법**

| 모델 | 방식 |
|---|---|
| BERT | [MASK]×|E| 삽입 → 마스크 토큰 평균 로그확률 |
| GPT-2 | 후보 첫 토큰 확률 |
| WKLM | P(e|C) 그대로. 학습 목적함수와 동일 |

WKLM은 자기 학습 목적함수와 형식이 동일한 과제에서 평가받는다.

## 10. Fact Completion 결과 (Table 1)

| Relation Name | # of Candidates | # of Answers | BERT-base | BERT-large | GPT-2 | Ours |
|---|---|---|---|---|---|---|
| HasChild (P40) | 906 | 3.8 | 9.00 | 6.00 | 20.5 | **63.5** |
| NotableWork (P800) | 901 | 5.2 | 1.88 | 2.56 | 2.39 | **4.10** |
| CapitalOf (P36) | 820 | 2.2 | 1.87 | 1.55 | 15.8 | **49.1** |
| FoundedBy (P112) | 798 | 3.7 | 2.44 | 1.93 | 8.65 | **24.2** |
| Creator (P170) | 536 | 3.6 | 4.57 | 4.57 | 7.27 | **9.84** |
| PlaceOfBirth (P19) | 497 | 1.8 | 19.2 | **30.9** | 8.95 | 23.2 |
| LocatedIn (P131) | 382 | 1.9 | 13.2 | 52.5 | 21.0 | **61.1** |
| EducatedAt (P69) | 374 | 4.1 | 9.10 | 7.93 | 11.0 | **16.9** |
| PlaceOfDeath (P20) | 313 | 1.7 | **43.0** | 42.6 | 8.83 | 26.5 |
| Occupation (P106) | 190 | 1.4 | 8.58 | **10.7** | 9.17 | **10.7** |
| **Average Hits@10** | − | − | 11.3 | 16.1 | 16.3 | **28.9** |

- WKLM이 10개 중 8개 관계에서 최고다. 평균은 BERT-large의 약 1.8배다
- 평균에서 GPT-2가 BERT를 앞선다. fact completion은 왼쪽 문맥만으로 예측해야 하는데 BERT는
  양방향 전제다
- BERT가 이기는 곳은 지리 관계다. 위키피디아에서 위치 엔티티가 문장 끝에 오는 경향이 있다
  (`Obama was born in Honolulu, Hawaii.`)
- 인명이 정답인 관계에서는 BERT가 GPT-2·WKLM 모두에 밀린다
- 강건성을 보면 BERT는 후보 집합 크기·정답 개수에 강하게 상관한다. WKLM은 훨씬 덜 민감하다

## 11. Open-domain QA 설정

| Dataset | Train | Valid | Test | Example Questions |
|---|---|---|---|---|
| WebQuestions | 3778 | − | 2032 | Who plays Stewie Griffin on Family Guy? |
| TriviaQA | 87291 | 11274 | 10790 | What is the Japanese share index called? |
| SearchQA | 99811 | 13893 | 27247 | Hero several books 11 discover's wizard? |
| Quasar-T | 37012 | 3000 | 3000 | Which vegetable is a Welsh emblem? |

파이프라인은 질문에서 IR 검색으로 문단 k개를 가져오고, 스팬 추출 점수와 Paragraph Ranker 점수를
선형 결합해 재랭킹한다. 학습은 distant supervision이라 검색된 문단 내 어떤 스팬이든 원 정답과
일치하면 정답으로 취급한다. 검색은 WebQ가 DrQA TF-IDF 상위 5개 문서, Quasar-T가 Lucene,
SearchQA·TriviaQA가 검색엔진 랭킹 문단이다.

**SQuAD 사전 실험 (Table 3)**

| Model | EM | F1 |
|---|---|---|
| Google's BERT-base | 80.8 | 88.5 |
| Google's BERT-large | 84.1 | 90.9 |
| Our BERT-base | 83.4 | 90.5 |
| WKLM (base) | 84.3 | 91.3 |

자체 BERT가 강한 이유는 2M 대 1M updates다. 이후 비교는 자체 BERT와만 수행한다. SQuAD는
비-엔티티 스팬 정답이 많은데도 +0.8 F1이다.

## 12. [미수록] Open-domain QA 결과

덱의 42쪽이다. 강의 영상에서 이 장을 넘어가 내용을 받지 못했다. 앞뒤 흐름으로 보면 원논문
Table 4(네 데이터셋 EM/F1)에 해당한다. 발표자는 41쪽에서 데이터셋별 엔티티 정답 비율을 언급한 뒤
곧바로 43쪽 FIGER로 넘어갔다.

원논문에서 확인한 값은 [S22-2 8절](S22-2-방향이-뒤집힌-회차.md)에 적어뒀다.

## 13. Fine-grained Entity Typing 결과 (Table 5)

| Model | Acc | Ma-F1 | Mi-F1 |
|---|---|---|---|
| LSTM + Hand-crafted (Inui et al., 2017) | 57.02 | 76.98 | 73.94 |
| Attentive + Hand-crafted (Inui et al., 2017) | 59.68 | 78.97 | 75.36 |
| BERT baseline (Zhang et al., 2019) | 52.04 | 75.16 | 71.63 |
| ERNIE (Zhang et al., 2019) | 57.19 | 75.61 | 73.39 |
| Our BERT | 54.53 | 79.57 | 74.74 |
| **WKLM** | **60.21** | **81.99** | **77.00** |

- BERT를 순진하게 적용하면 수작업 피처 모델보다 못하다 (52.04 대 59.68)
- ERNIE는 +5.15 개선이지만 여전히 수작업 피처 모델에 못 미친다
- WKLM은 더 강한 BERT에서 출발했는데도 +5.68 개선으로 SOTA를 달성한다

개선폭을 나란히 보면 ERNIE가 52.04에서 57.19로 +5.15이고, WKLM이 54.53에서 60.21로 +5.68이다.
수작업 피처 모델 기준선은 59.68이다.

> 두 개선폭은 서로 다른 BERT 베이스라인 위에서 측정된 값이라는 주의가 슬라이드에 함께 붙어 있다.

저자 주장은 텍스트에서 직접 지식을 배우는 쪽이 KB 임베딩 주입(ERNIE)보다 효과적이라는 것이다.

## 14. Ablation — MLM의 역할 (Table 6)

동기는 RoBERTa(Liu 2019b)가 BERT를 더 오래 학습하는 것만으로 성능이 오른다고 보고한 데 있다.
WKLM의 이득은 목적함수에서 오는가, 학습량에서 오는가.

| Model | SQuAD EM | SQuAD F1 | TriviaQA EM | TriviaQA F1 | Quasar-T EM | Quasar-T F1 | FIGER Acc |
|---|---|---|---|---|---|---|---|
| Our BERT | 83.4 | 90.5 | 48.7 | 53.2 | 40.4 | 46.1 | 54.53 |
| **WKLM** | 84.3 | **91.3** | **52.2** | **56.7** | **43.7** | **49.9** | **60.21** |
| WKLM without MLM | 80.5 | 87.6 | 48.2 | 52.5 | 42.2 | 48.1 | 58.44 |
| WKLM with 15% masking | 84.1 | 91.0 | 51.0 | 55.3 | 42.9 | 49.0 | 59.68 |
| Our BERT + 1M MLM updates | **84.4** | 91.1 | 52.0 | 56.3 | 42.3 | 48.2 | 54.17 |

- MLM은 필수다. without MLM은 SQuAD에서 급락한다 (80.5 / 87.6)
- 마스킹 비율은 5%가 15%보다 낫다. 마스크가 많으면 문맥이 손상되어 지식 학습에 노이즈가 된다
- +1M MLM은 SQuAD 최고(84.4)지만 FIGER는 54.17로 제자리다. Our BERT 54.53보다 낮다

결정적 증거는 여기에 있다. 추가 MLM 학습은 엔티티 지식을 늘리지 못한다. WKLM의 이득은 학습량이
아니라 목적함수에서 온다. WKLM은 엔티티 관련 과제에서 MLM을 보완하는 레시피로 기능한다.

## 15. WKLM 정리 & 한계

사전학습에 최소한의 엔티티 정보(Wikidata 타입 라벨 + 앵커 링크)만 사용하고, 다운스트림에서 추가
계산·메모리·구조 변경이 없다. 비정형 자연어에서 엔티티 레벨 지식을 직접 학습하는 것이 가능하다.

| 한계 | 내용 |
|---|---|
| 지식이 파라미터에 갇힘 | 무엇을 아는지 열람·감사 불가. 사실이 바뀌면 재학습해야 한다 |
| 위키피디아 의존 | 앵커 링크 품질에 기대므로 도메인 코퍼스에 그대로 적용하기 어렵다 |
| 평가의 순환성 | fact completion 형식이 학습 목적함수와 유사해 유리할 수 있다 |
| 타입 정의의 임의성 | P31에 타입이 여러 개면 무작위 선택이라 온톨로지 관점에서 거친 처리다 |
| 관계 지식은 간접적 | 엔티티를 판별할 뿐 관계 자체를 명시적으로 학습하지 않는다 |

규제 산업(반도체·의료)에서는 첫 번째 한계가 치명적 제약이다.

## 16. 직접 비교 — COMET vs WKLM

| 축 | COMET | WKLM |
|---|---|---|
| 지식의 방향 | LM → KG (생성) | 텍스트 → LM (내재화) |
| 백본 | GPT (단방향) | BERT-base (양방향) |
| 감독 신호 | 강함. gold 트리플 710k / 100k | 약함. 치환 여부 이진 라벨 |
| KG 의존도 | seed 트리플 필수 | 타입 라벨만 (P31) |
| 학습 목적 | 조건부 생성 (sr → o) | 이진 판별 + MLM |
| 산출물 | 명시적 트리플 (검사 가능) | 파라미터 (검사 불가) |
| 지식 갱신 | 새 트리플 생성·삭제로 가능 | 재학습 필요 |
| 검증 가능성 | 사람이 튜플 단위로 검증 | 프로빙 과제로 간접 추정만 |
| 사실성 보장 | 없다. plausible ≠ true | 없다. 그러나 실제 문서 기반이다 |
| 스키마 제약 | 없다 | 없다 |
| 다운스트림 | KG 확장 자체가 목적 | QA · entity typing 등 |
| 평가 방식 | 인간 평가 중심 | 자동 벤치마크 중심 |
| 대표 성과 | 사람 평가 77.5% / 91.7% | Hits@10 28.9 · QA +2.7 F1 · FIGER SOTA |

COMET은 지식을 꺼내 놓고 검증을 사람에게 맡기고, WKLM은 지식을 집어넣고 성능으로 존재를
증명한다.

## 17. Ch.6 모델 정리

| 세션 | 모델 | 지식 소스 | 주입 / 생성 방식 | 다운스트림 오버헤드 |
|---|---|---|---|---|
| S19 | ERNIE | KG 엔티티 임베딩 | 어텐션 레이어에 융합 | 엔티티 링킹 + KB 임베딩 |
| S19 | KnowBERT | 여러 KB | KB 어텐션 레이어 삽입 | 구조 변경 |
| S20 | K-BERT | KG 트리플 | 문장에 트리플 삽입 (soft-position) | 트리플 주입 파이프라인 |
| S20 | KEPLER | KG 트리플 | KE loss + MLM 공동 학습 | 없음 |
| S21 | K-Adapter | 다양한 지식 | 어댑터 모듈 (LM 고정) | 어댑터 |
| S21 | CoLAKE | word-knowledge 통합 그래프 | 통합 그래프 사전학습 | 그래프 구성 |
| S22 | WKLM | 텍스트 + 타입 라벨 | 엔티티 치환 판별 | 없음 |
| S22 | COMET | seed 트리플 | 방향 역전. KG를 출력 | 해당 없음 |

**흐름 정리**

| | 질문 | 답 |
|---|---|---|
| S19 ~ S21 | "어디에 어떻게 넣을까" | 레이어 → 문장 → 어댑터 |
| WKLM | "굳이 넣어야 하나?" | 텍스트에 이미 있는데. 질문 변경 |
| COMET | "꺼내는 게 목적이면?" | 넣는 게 목적이 아니다. 축 변경 |

---

## 관련 문서

- 앞 본편 — [S22A COMET: 언어모델로 KG를 만든다](S22A-COMET-언어모델로-KG를-만든다.md)
- 부록 — [S22-1 읽는 데 필요한 것들](S22-1-읽는-데-필요한-것들.md) ·
  [S22-2 방향이 뒤집힌 회차](S22-2-방향이-뒤집힌-회차.md)
- Ch.6 종합 — [01 Ch.4~6 무엇이 남고 무엇이 밀렸나](01-Ch4-6-무엇이-남고-무엇이-밀렸나.md)
