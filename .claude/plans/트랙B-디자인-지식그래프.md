# 트랙 B — 디자인 지식그래프 + MCP

**이 문서는 [`Junseop-Shin/design-kg`](https://github.com/Junseop-Shin/design-kg) 레포의 [`docs/plan.md`](https://github.com/Junseop-Shin/design-kg/blob/main/docs/plan.md)로 이식했다.** (2026-09-10)

이식하면서 개정한 것:

- 평가 장치가 "컴포넌트 1개 위반 수 비교"에서 **랜딩페이지 A/B/C 비교**(테마만 교체 / 자연어 지시 MCP 없이 / MCP 연결)로 바뀌었다. 판정은 재현성 · 출처 추적 · 기본기 · 눈.
- `Quality` 노드(형용사 → 속성 · 방향)가 추가돼 MCP 툴이 4개 → 5개.
- 선행 단계로 **my-ui-lib에 디자인 테마 축**(`data-theme` × `data-mode`)이 생긴다. `default` + `finance` + tweakcn 4종.
- 착수 순서는 영상 추출(3단계)이 먼저. my-ui-lib 테마 작업(1~2단계)은 독립이라 병렬 가능.

로컬 경로: `Projects/design-kg/` (개별 git repo, Work에서는 gitignore).
