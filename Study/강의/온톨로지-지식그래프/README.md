# 온톨로지 & 지식그래프 (Ontology & Knowledge Graph)

> 강의 학습 정리 — **회차(세션)마다 문서 하나씩.** 파일명 `SNN-주제.md`.

## 강의 정보

| | |
|---|---|
| 주제 | 온톨로지 설계 · 평가 · 지식그래프 구축 · LLM 결합 |
| 구성 | 11챕터 / 37세션 / 10주 (Day 1~19) |
| 강의 | [YouTube 재생목록](https://www.youtube.com/watch?v=f0WV7b3lGqM&list=PLFHGWfB_kmrs) · DSBA Lab Study |
| 주요 문헌 | Gruber(1995) 설계 5원칙 · FAIR(2020) 출판 4단계 |
| 실습 | OWL 설계, RDF 파이프라인, PyKEEN, DeepOnto, OntoGPT/SPIRES |
| 연관 실무 | [제조 암묵지 온톨로지 과제](../../../Work_History/2026-06-온톨로지-파이프라인-학습.md) · [`Projects/ontology-pipeline`](../../../Projects/ontology-pipeline/) |

**왜 듣는가** — 실무 과제에서 파이프라인(정제 → RML 매핑 → SHACL 검증 → GraphDB 적재 → SPARQL)을
한 바퀴 완주했지만, 현장 숙련자의 암묵지를 그래프로 옮기는 지점에서 막혔다. 이 강의의 Ch.3(평가 방법론),
Ch.7~8(LLM 기반 구축), Ch.11(이미지·멀티모달)이 그 막힌 지점을 직접 겨냥한다.

## 회차별 정리 현황

| 주차 | 일차 | 챕터 | 세션 | 내용 | 문서 | 상태 |
|---|---|---|---|---|---|---|
| 1주 | Day 1 | Ch.1 오리엔테이션 | S01 | 오리엔테이션 + 기본 개념 & 실사례 | [S01-기본개념과-실사례](S01-기본개념과-실사례.md) | ✅ |
| | | ↳ 부록 | S01-1 | 연관 개념의 층위 — SKOS·OWL · Semantic Web · Schema | [S01-1-연관-개념의-층위](S01-1-연관-개념의-층위.md) | ✅ |
| | | Ch.2-1 전통 구축 방법론 | S02 | 설계 원칙 | [S02-설계-원칙](S02-설계-원칙.md) | ✅ |
| | | ↳ 부록 | S02-1 | 두 계보(W3C vs 운영 온톨로지)와 '발견'의 세 층 | [S02-1-두-계보와-발견-메커니즘](S02-1-두-계보와-발견-메커니즘.md) | ✅ |
| | Day 2 | Ch.2-1 | S03 | 텍스트 기반 온톨로지 자동 추출 | — | ⬜ |
| | | Ch.2-1 | S04 | 온톨로지 엔지니어링 방법론 | — | ⬜ |
| 2주 | Day 3 | Ch.2-2 도메인 구축 사례 | S05 | 도메인 온톨로지 구축 사례 ① | — | ⬜ |
| | | Ch.2-2 | S06 | 도메인 온톨로지 구축 사례 ② | — | ⬜ |
| | Day 4 | Ch.3 평가 방법론 | S07 | 구조적·품질 평가 (OQuaRE) | — | ⬜ |
| | | Ch.3 | S08 | 기능적·의미적 평가 (CQ 기반) | — | ⬜ |
| 3주 | Day 5 | 실습 ① | S09 | OWL 온톨로지 설계 실습 | — | ⬜ |
| | | 실습 ② | S10 | RDF 데이터 파이프라인 실습 | — | ⬜ |
| | Day 6 | Ch.4 온톨로지↔KG | S11 | 지식그래프 기초 개념 | — | ⬜ |
| | | Ch.4 | S12 | KG 구축에서 온톨로지의 역할 | — | ⬜ |
| 4주 | Day 7 | Ch.5 ML/DL | S13 | KG 임베딩 기초 (TransE · DistMult) | — | ⬜ |
| | | Ch.5 | S14 | GNN 기반 KG 표현 (R-GCN · CompGCN) | — | ⬜ |
| | Day 8 | Ch.5 | S15 | 온톨로지 임베딩 (OWL2Vec* · EL) | — | ⬜ |
| | | Ch.5 | S16 | 딥러닝 기반 온톨로지 정렬 (OntoEA · BERTMap) | — | ⬜ |
| 5주 | Day 9 | 실습 ③ | S17 | KG 임베딩 실습 — PyKEEN | — | ⬜ |
| | | 실습 ④ | S18 | 온톨로지 임베딩·정렬 실습 — DeepOnto | — | ⬜ |
| | Day 10 | Ch.6 언어모델+KG | S19 | 엔티티 임베딩 직접 주입 (ERNIE · KnowBERT) | — | ⬜ |
| | | Ch.6 | S20 | KG 구조 통합 & 공동 학습 (K-BERT · KEPLER) | — | ⬜ |
| 6주 | Day 11 | Ch.6 | S21 | 파라미터 효율적 주입 & 통합 그래프 (K-Adapter · CoLAKE) | — | ⬜ |
| | | Ch.6 | S22 | 상식 지식 생성 & 약한 지도학습 (COMET · Pretrained Encyclopedia) | — | ⬜ |
| | Day 12 | Ch.7 LLM 기반 온톨로지 구축 | S23 | LLMs4OL | — | ⬜ |
| | | Ch.7 | S24 | CQ 기반 Human-in-the-loop 파이프라인 | — | ⬜ |
| 7주 | Day 13 | Ch.7 | S25 | 멀티에이전트 온톨로지 생성 | — | ⬜ |
| | | Ch.7 | S26 | Schema-grounded KG 구축 | — | ⬜ |
| | Day 14 | Ch.7 | S27 | KGOE 가속화 & 엔터프라이즈 온톨로지 | — | ⬜ |
| | | Ch.8 LLM으로 KG 구축 | S28 | 관계 추출 기반 KG 생성 (REBEL) | — | ⬜ |
| 8주 | Day 15 | Ch.8 | S29 | LLM 기반 KG 구축·추론 서베이 | — | ⬜ |
| | | Ch.8 | S30 | LLM 기반 개체 정렬 (LLMEA) | — | ⬜ |
| | Day 16 | Ch.8 | S31 | LLM + KG 통합 로드맵 (Pan et al.) | — | ⬜ |
| | | Ch.9 LLM + KG 활용 | S32 | 온톨로지 기반 LLM 파인튜닝 (OntoTune) | — | ⬜ |
| 9주 | Day 17 | Ch.10 시계열 | S33 | SSN/SOSA & W3C Time | — | ⬜ |
| | | Ch.10 | S34 | IoT·스마트빌딩 적용 (SOSA · Brick) | — | ⬜ |
| | Day 18 | Ch.11 이미지 | S35 | 시각 관계 KG & 씬 그래프 | — | ⬜ |
| | | Ch.11 | S36 | 이미지 분류 온톨로지 & 멀티모달 KG | — | ⬜ |
| 10주 | Day 19 | 실습 ⑤ | S37 | OntoGPT/SPIRES Python 실습 ① | — | ⬜ |

*(목차 슬라이드가 S37에서 잘려 있어 이후 세션은 미확인)*

## 정리 규칙

- 파일명: `SNN-주제.md` — 세션 번호 prefix로 순서 고정
- 회차당 문서 1개가 기본. 분량에 따라 조정한다
  - 여러 회차가 짧고 이어지면 묶고 `SNN-SNN-주제.md`
  - 한 회차가 크면 쪼개고 `SNNA-주제.md` · `SNNB-주제.md` (부록 `SNN-N`과 구분됨)

**본편과 부록을 나눈다.**

| | 담는 것 |
|---|---|
| 본편 `SNN-주제.md` | **강의 내용에 충실하게.** 자료에 있는 것만. 절 구성도 강의 흐름을 따른다 |
| 부록 `SNN-N-주제.md` | 자료 밖에서 나온 것 — 대화로 정리한 인사이트, 다른 계보/개념과의 비교, 검증할 가설 |

- 부록은 문서 상단에 `성격` 메모로 출처(어느 절에서 갈라져 나왔는지)를 남긴다
- 본편에서 부록으로 링크를 걸어 흐름이 끊기지 않게 한다
- 본편 안에 자료 밖 내용을 넣어야 할 때는 인용 블록(`>`)으로 표시해 구분한다
- 재사용성 높은 개념은 `Study/기타/아키텍처/` 등 주제 카테고리로 승격하고 상호 링크
  (강의 노트 = 수강 흐름 / 카테고리 = 주제 검색)
- 실무 과제와 겹치는 내용은 `Projects/ontology-pipeline/docs/`를 링크하고 중복 서술하지 않는다

## 관련 문서

- [`Projects/ontology-pipeline/docs/1.core-concepts.html`](../../../Projects/ontology-pipeline/docs/1.core-concepts.html) — RDF vs LPG, OWA/CWA, 표준 온톨로지(BFO·IOF·SOSA)
- [`Projects/ontology-pipeline/docs/3.ontology-design.html`](../../../Projects/ontology-pipeline/docs/3.ontology-design.html) — 실제 도메인 온톨로지 설계
- [`Projects/ontology-pipeline/docs/4-1.cleaning-mapping-validation.html`](../../../Projects/ontology-pipeline/docs/4-1.cleaning-mapping-validation.html) — RML 매핑 · SHACL 검증
- [Work_History — 파이프라인은 완주했는데, 공장에서 명장을 만나고 다시 막막해졌다](../../../Work_History/2026-06-온톨로지-파이프라인-학습.md)
