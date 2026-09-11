# /resume — 이력서 수정 및 PDF 생성

이력서를 고칠 때는 **항상 `_private/build_resume.py` 만 고친다.** docx·pdf는 산출물이므로 직접 편집하지 않는다.
수정 후에는 실명·공개용 docx를 다시 뽑고, 공개용을 기반으로 날짜가 박힌 PDF를 만든다.

---

## 파일 구조

| 파일 | 성격 |
|---|---|
| `_private/build_resume.py` | 유일한 원본. 여기만 수정한다 |
| `_private/이력서_신준섭.docx` | 실명 버전 (제출용, `_private` 밖으로 내보내지 않는다) |
| `_private/이력서_신준섭_공개용.docx` | 익명 버전 (공개 사이트·PDF의 기반) |
| `_private/이력서_신준섭_{YYYY-MM-DD}.pdf` | 공개용 docx에서 뽑은 배포본 |

`--public` 분기가 고객사명을 업종 표현으로 바꾸고 전화번호·생년을 뺀다. 새 고객사명을 쓸 일이
생기면 상단 `CLIENT` 처리 방식을 따라 분기에 넣는다. 실명을 본문에 직접 적지 않는다.

---

## 절차

### 1. 원본 수정

`_private/build_resume.py` 에서 해당 `project(...)` · `bullets([...])` 블록을 고친다.

- 수치는 출처가 있는 것만 쓴다. 추정치를 단정형으로 적지 않는다
- 고객사명은 어떤 경우에도 평문으로 넣지 않는다 (`CLIENT` 변수 경유)
- 사내 레포명·리소스명·계정명·구독 식별자·테넌트 ID를 넣지 않는다
- 한 프로젝트의 불릿이 10개를 넘으면 묶거나 쪼갤 것을 먼저 제안한다

### 2. docx 재생성 — 두 버전 모두

```bash
cd ~/Documents/Work/_private
./venv/bin/python build_resume.py            # 실명
./venv/bin/python build_resume.py --public   # 익명
```

`venv` 를 쓴다. 시스템 python 에는 `python-docx` 가 없다.

### 3. 익명화 검증

검사어를 이 문서에 적지 않는다. 이 레포는 공개이므로 고객사명을 평문으로 두면 검증
스크립트가 그 자체로 유출이 된다. 대신 `build_resume.py` 의 `CLIENT` 분기에서 실명을
런타임에 읽어 공개용 산출물에 남아 있는지 본다.

스크립트를 파일로 저장해 실행한다. 셸 heredoc 안에 파이썬 heredoc을 중첩하면 깨진다.

```bash
cd ~/Documents/Work/_private
cat > /tmp/check_resume.py <<'SCRIPT'
import re
from docx import Document

src = open("build_resume.py", encoding="utf-8").read()
m = re.search(r'CLIENT\s*=\s*"(.+?)"\s*if\s*PUBLIC\s*else\s*"(.+?)"', src)
anon, real = m.group(1), m.group(2)

for f, label in [("이력서_신준섭_공개용.docx", "공개용"), ("이력서_신준섭.docx", "실명")]:
    t = "\n".join(p.text for p in Document(f).paragraphs)
    print(f"[{label}] 실명 노출 {'있음' if real in t else '없음'}"
          f" / 익명 표현 {'있음' if anon in t else '없음'}")
SCRIPT
./venv/bin/python /tmp/check_resume.py
```

기대값은 공개용이 "실명 노출 없음 / 익명 표현 있음", 실명 버전이 그 반대다.
공개용에서 실명이 잡히면 `CLIENT` 를 거치지 않고 본문에 직접 적은 곳이 있다는 것이므로
그 자리를 찾아 고친다.

### 4. PDF 생성

공개용 docx를 Pages로 열어 내보낸다. 변환용 LibreOffice·pandoc은 설치돼 있지 않다
(검증용 `pdftotext` 는 있다 — 4-1 참고).
기존 PDF도 Pages로 뽑은 것이라 이 경로를 유지해야 레이아웃이 일관된다.

**`with timeout` 을 반드시 감싼다.** 없으면 기본 AppleEvent 제한(약 60초)에 걸려
`-1712` 오류로 실패하고 PDF가 만들어지지 않는다. 변환에 그보다 오래 걸린다.

```bash
DATE=$(date +%F)
osascript <<APPLESCRIPT
set srcFile to POSIX file "/Users/js/Documents/Work/_private/이력서_신준섭_공개용.docx"
set outFile to POSIX file "/Users/js/Documents/Work/_private/이력서_신준섭_${DATE}.pdf"
tell application "Pages"
    activate
    with timeout of 600 seconds
        set theDoc to open srcFile
        delay 3
        export theDoc to outFile as PDF
        close theDoc saving no
    end timeout
end tell
APPLESCRIPT
ls -la ~/Documents/Work/_private/이력서_신준섭_${DATE}.pdf
```

주의할 것

- 명령이 오래 걸리므로 백그라운드로 돌리고 완료 알림을 기다린다
- Pages 자동화 권한 요청이 뜨면 사용자에게 승인을 요청한다. 창이 잠깐 뜨는 것은 정상이다
- 실패하면 Pages에 문서가 열린 채로 남는다. `tell application "Pages" to close document 1 saving no`
  로 정리한 뒤 재시도한다
- `delay` 를 줄이면 변환이 끝나기 전에 export가 실행돼 빈 PDF가 나올 수 있다

### 4-1. PDF 검증

배포되는 것은 PDF다. docx만 검증하고 넘어가지 않는다. `pdftotext`(poppler)는 설치돼 있다.

```bash
cd ~/Documents/Work/_private
NEW="이력서_신준섭_$(date +%F).pdf"
echo "페이지 수: $(mdls -name kMDItemNumberOfPages -raw "$NEW")"
T=$(pdftotext "$NEW" -)
echo "$T" | head -3          # 헤더가 의도대로 나왔는지
# 공개용에 남으면 안 되는 것 — 전화번호·생년 등 실명 버전 전용 항목
echo "$T" | grep -c "010-"   # 0 이어야 한다
```

고객사명은 `CLIENT` 실명을 직접 적지 말고 3단계 스크립트를 재사용해 비교한다.
페이지 수가 이전 판보다 늘었으면 사용자에게 알리고 줄일지 확인한다.

### 5. 포트폴리오 사이트 반영 (선택)

공개 사이트의 이력서 다운로드는 `Projects/profile/next/public/resume.pdf` 다.
`about.ts` 의 `resumePath` 가 이 경로를 가리킨다.

```bash
cp ~/Documents/Work/_private/이력서_신준섭_$(date +%F).pdf \
   ~/Documents/Work/Projects/profile/next/public/resume.pdf
```

profile 레포는 별도 git 저장소다. 브랜치를 따로 파고 PR로 올린다 — main에 직접 커밋하지 않는다.

---

## 같이 갱신할 것

이력서 내용을 고치면 아래도 어긋나지 않는지 본다. 세 곳의 사실관계가 갈리면 신뢰가 깎인다.

| 대상 | 무엇 |
|---|---|
| `Projects/profile/next/data/projects.ts` | 같은 프로젝트의 항목 |
| `Projects/profile/next/data/about.ts` | 자격증, 강점 서술, 소개 문단 |
| `Work_History/` | 해당 시기의 글 |

`_private/` 와 `Projects/*/` 는 gitignore 대상이라 Work 레포 커밋에는 잡히지 않는다.
Work_History만 Work 레포 소속이고, **이 레포는 공개**이므로 내부 식별자를 넣지 않는다.
