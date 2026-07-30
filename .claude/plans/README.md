# Plans

진행 예정 작업의 설계 문서. 구현 전 단계이며, 각 문서는 독립된 spec → plan → 구현 사이클을 가진다.

## 배경

참고 자료 4건에서 출발한 계획을 4개 트랙으로 분해했다. 세 건([Schema 하네스](https://www.aitimes.com/news/articleView.html?idxno=212889) · [graphify](https://news.hada.io/topic?id=31743) · [brunch: 지어내지 말고 찾아 쓰라](https://brunch.co.kr/@kim-yezi/48))이 같은 논지로 수렴한다 — **모델이 아니라 하네스가 성능을 만든다.** 나머지 한 건([shadcn Base UI 전환](https://news.hada.io/topic?id=31163))은 별건이다.

## 트랙

| 트랙 | 문서 | 상태 | 요지 |
|---|---|---|---|
| A | — | 진행 중 | 온톨로지 강의 정리 (S01/37 — [`Study/강의/온톨로지-지식그래프/`](../../Study/강의/온톨로지-지식그래프/README.md)). 실무: `Projects/ontology-pipeline` + `Work_History/2026-06-온톨로지-파이프라인-학습.md` |
| B | [디자인 지식그래프](./트랙B-디자인-지식그래프.md) | 설계 완료 | UI/UX 지식을 그래프화해 LLM에 붙이는 조회·검증 하네스 |
| C | [Base UI 전환 + 툴체인 최신화](./트랙C-base-ui-전환.md) | ✅ **완료** (v1.0.0) | Radix 12개 + sonner → `@base-ui/react` 1개. 컴포넌트 33 → 57, 툴체인 5종 메이저 업그레이드 |
| D | [HTML 편집 익스텐션](./트랙D-html-편집-익스텐션.md) | 설계 완료 | 브라우저에 열린 HTML을 그 자리에서 고치고 저장하는 크롬 익스텐션 |

## 순서

```
C ─────────────────► D(4단계)     C가 my-ui-lib DOM 구조를 바꾸므로
                                  스니펫 추출은 C 이후

B ─ ─ ─ ─ ─ ─ ─ ─ ─► D            연동 가능하나 엮지 않음.
                                  B의 2단계 판정 이후 재검토

```

**C를 먼저 한다.** 스코프가 가장 명확하고 D의 선행 조건이다. B와 D는 독립적으로 진행 가능하다.

각 트랙은 자기 문서 안에 진짜 판정 지점(go/no-go)을 하나씩 갖고 있다. B는 2단계의 위반 수 비교, C는 Chromatic diff, D는 편집 전후 HTML diff다.
