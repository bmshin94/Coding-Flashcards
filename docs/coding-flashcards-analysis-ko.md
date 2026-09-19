# Coding Flashcards 분석 & 활용 가이드 (한국어)

> 작성일: 2026-09-19
> 작성: Claude Code (카리나 페르소나) × @bmshin94
> 이 문서는 저장소 전체를 직접 분석한 뒤, 활용법과 수익화 전략까지 정리한 종합 가이드입니다.

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| **내 저장소 (fork)** | https://github.com/bmshin94/Coding-Flashcards |
| **원본 저장소 (upstream)** | https://github.com/ad-si/Coding-Flashcards |
| **릴리스 (덱 다운로드)** | https://github.com/ad-si/Coding-Flashcards/releases |
| anki-panky (MD→apkg 변환기) | https://github.com/kamalsacranie/anki-panky |
| Anki (학습 앱) | https://apps.ankiweb.net |
| Pandoc | https://pandoc.org |
| Tectonic (LaTeX 엔진) | https://tectonic-typesetting.github.io |
| Rust Book (Rust 덱 출처) | https://doc.rust-lang.org/book/ |
| AnkiWeb - Rust Flashcards | https://ankiweb.net/shared/info/1541471942 |

---

## 1. 프로젝트 한 줄 정의

> **마크다운으로 작성한 프로그래밍 암기카드 약 1,035장 + 이를 Anki 덱(`.apkg`)과 PDF로 자동 변환하는 빌드 파이프라인**

실행되는 애플리케이션이나 라이브러리가 아니라, **"학습 콘텐츠 저장소 + 자동화 빌드 시스템"** 입니다.

---

## 2. 저장소 구조

```
Coding-Flashcards/
├── readme.md                    프로젝트 소개
├── CLAUDE.md                    Claude Code 페르소나 지침 (이 저장소에서 추가)
├── makefile                     전체 빌드 + 통합 덱 생성
├── common.mk                    각 언어 폴더가 공유하는 빌드 규칙
├── flake.nix / flake.lock       Nix 개발환경 (툴 버전 고정)
├── .envrc                       direnv 연동 (`use flake`)
├── .gitignore                   빌드 산출물 제외
├── .github/workflows/
│   └── build-artifacts.yml      GitHub Actions 자동 빌드/릴리스
│
├── rust/                        557장 (Rust Book 전체 기반)
├── sqlite/                      166장
├── lua/                         109장
├── godot/                        81장
├── wolfram-language/             76장
└── c/                            46장
```

각 언어 폴더는 동일한 구조를 가집니다.

| 파일 | 역할 |
|---|---|
| `cards.md` | 실제 카드 내용 (유일한 원본 데이터) |
| `makefile` | `include ../common.mk` 단 한 줄 |
| `README.md` | 해당 덱 설명 |
| `images/`, `source/` | 일부 덱만 보유 (스크린샷, 예제 코드) |

> **설계 포인트** — 빌드 로직은 `common.mk` 한 곳에만 존재합니다. 새 언어를 추가하려면 폴더를 만들고 `cards.md` + 한 줄짜리 makefile만 넣으면 끝입니다.

### 덱별 카드 수

| 덱 | 카드 수 | 비중 |
|---|---:|---|
| Rust | ~557 | `████████████████████████████` |
| SQLite | ~166 | `████████` |
| Lua | ~109 | `█████` |
| Godot | ~81 | `████` |
| Wolfram Language | ~76 | `███` |
| C | ~46 | `██` |
| **합계** | **~1,035** | |

---

## 3. 카드 포맷 (`cards.md`)

```markdown
---
name: C Flashcards        ← 덱 이름 (YAML frontmatter)
---

---                       ← 카드 구분선

Make a variable unmodifiable.    ← 앞면 (질문)

. . .                            ← 앞/뒤 경계 마커

`const`                          ← 뒷면 (정답)

---                              ← 다음 카드
```

**핵심 규칙 두 가지**

- `---` (3개 이상의 하이픈) → 카드 구분선
- `. . .` → 카드 앞면과 뒷면의 경계

`. . .`는 원래 **Pandoc의 pause 문법**(슬라이드에서 "여기서 멈춤")인데, 이를 플래시카드의 앞/뒤 경계로 재활용한 것이 이 프로젝트의 영리한 설계입니다.

> ⚠️ Wolfram Language 덱은 구분선으로 하이픈 80개(`---...---`)를 사용합니다. 파서를 만들 때 `^-{3,}$` 정규식을 쓰면 모두 커버됩니다.

코드 블록을 포함한 카드 예시:

````markdown
---

What's the command to run a package?

. . .

```sh
cargo run
```

---
````

---

## 4. 빌드 파이프라인

```
cards.md (마크다운, 단일 원본)
    │
    ├──[ anki-panky ]──→ cards.apkg              (Anki 덱)
    │                    cards-with-name.apkg    (덱 이름 라벨 포함)
    │
    └──[ pandoc + tectonic ]──→ cards.pdf        (130mm×130mm beamer 슬라이드)
```

| 도구 | 역할 |
|---|---|
| `anki-panky` | 마크다운 → Anki `.apkg` 변환 (Haskell 제작, v0.0.0.7) |
| `pandoc` | 마크다운 → beamer 중간 포맷 |
| `tectonic` | LaTeX → PDF 렌더링 |
| Hasklug Nerd Font Mono | 코드 블록 전용 폰트 |

PDF는 `papersize={130mm,130mm}` 정사각형으로 설정되어 실제 카드처럼 보입니다.

### 통합 덱 생성 (루트 `makefile`의 awk 스크립트)

1. 각 `cards.md`의 YAML frontmatter 제거
2. 이미지 경로 보정 (`images/` → `rust/images/`)
3. 각 카드 하단에 원래 덱 이름을 회색 작은 글씨로 삽입 (HTML/LaTeX 양쪽 대응)

### 주요 make 타겟

```bash
make help              # 사용 가능한 타겟 목록
make build             # 6개 언어 전체 빌드
make build-combined    # 통합 덱 (.apkg + .pdf)
make test              # nix flake check + 각 덱 검증
make clean             # 생성물 삭제

cd rust && make build  # 특정 언어만 빌드
```

---

## 5. Nix 개발환경

`flake.nix`는 `anki-panky` 릴리스 바이너리를 SHA256 해시까지 고정해 가져옵니다.

```nix
version = "0.0.0.7";
sha256 = "XF/7+4AvJ2VBJP8lO/199HrJuvKHMOWqfGNB99oEs84=";  # ubuntu
sha256 = "/pZTo3OzIbtAqZC0GgT2SngxD15SoDJYsj5sGq+hrV4=";  # macOS
```

- 지원 시스템: `x86_64-darwin`, `x86_64-linux`, `aarch64-darwin`
- devShell 구성: `anki-panky`, `nerd-fonts.hasklug`, `pandoc`, `tectonic`
- `LD_LIBRARY_PATH`에 `gmp`, `zlib` 연결

**효과**: 로컬과 CI가 100% 동일한 툴체인으로 동작합니다. "내 컴에선 되는데" 문제가 구조적으로 발생하지 않습니다.

---

## 6. CI/CD (GitHub Actions)

`.github/workflows/build-artifacts.yml`

| 항목 | 내용 |
|---|---|
| 트리거 | `main` 푸시 / `v*.*.*` 태그 / PR / 수동 실행 |
| 실행 환경 | `macos-14` |
| 전략 | 6개 언어 매트릭스 병렬 빌드, `fail-fast: false` |
| 별도 job | `combined` — 통합 덱 생성 |
| 릴리스 | `v` 태그 시 `.apkg`/`.pdf` 자동 첨부 (`softprops/action-gh-release@v2`) |
| 권한 | `contents: write` |

폰트 처리 단계가 특이합니다. `nix develop --command true`로 dev-shell 입력을 `/nix/store`에 실체화한 뒤, Hasklug 폰트 파일을 `~/Library/Fonts`로 복사해 tectonic이 인식하게 만듭니다.

> ⚠️ 이 폰트 처리 방식은 macOS 경로(`~/Library/Fonts`)에 의존하므로, Linux에서 PDF까지 빌드하려면 `~/.local/share/fonts` 등으로 경로 수정이 필요합니다. `.apkg`만 필요하다면 무관합니다.

---

## 7. 설치 및 사용법

### 7.1 학습만 하고 싶다면 (권장, 5분)

빌드가 필요 없습니다. 완성된 결과물이 릴리스에 있습니다.

1. **Anki 설치** — https://apps.ankiweb.net (PC/안드로이드 무료, iOS 유료)
2. **덱 다운로드** — https://github.com/ad-si/Coding-Flashcards/releases
   - `cards-<언어>.apkg` — 언어별 덱
   - `cards-combined.apkg` — 6개 통합 덱
   - `cards-<언어>.pdf` — PDF 버전
3. `.apkg` 더블클릭 또는 Anki에서 `파일 → 가져오기`

> 💡 AnkiWeb 계정으로 동기화하면 PC ↔ 모바일 학습 진도가 이어집니다.

### 7.2 직접 빌드 (Nix 방식, 프로젝트 공식)

```bash
# 1. Nix 설치
curl --proto '=https' --tlsv1.2 -sSf -L \
  https://install.determinate.systems/nix | sh -s -- install

# 2. Flakes 활성화 (구버전 Nix인 경우)
mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf

# 3. 클론 & 진입
git clone https://github.com/bmshin94/Coding-Flashcards.git
cd Coding-Flashcards

# 4. 개발 환경 진입
nix develop

# 5. 빌드
make build
```

**direnv 연동** (`.envrc`에 `use flake` 존재)

```bash
brew install direnv   # 또는 apt install direnv
direnv allow          # 폴더 진입 시 환경 자동 활성화
```

### 7.3 Nix 없이 빌드 (비권장)

```bash
# anki-panky를 릴리스에서 직접 다운로드
# https://github.com/kamalsacranie/anki-panky/releases
brew install pandoc tectonic
# Hasklug Nerd Font 수동 설치 필요
anki-panky cards.md   # → cards.apkg
```

### 7.4 새 언어 덱 추가

```bash
mkdir react
echo "include ../common.mk" > react/makefile
# react/cards.md 작성
```

그리고 루트 `makefile`과 `.github/workflows/build-artifacts.yml`의 matrix에 `react`를 추가합니다.

---

## 8. 자주 묻는 질문 정리

### Q. 플러그인인가? Skill인가? MCP인가?

**셋 다 아닙니다.**

| 구분 | 해당 여부 | 근거 |
|---|---|---|
| 플러그인 | ❌ | 붙일 호스트 애플리케이션이 없는 독립 저장소 |
| Claude Skill | ❌ | `SKILL.md`, `.claude/skills/` 없음 |
| MCP 서버 | ❌ | 서버 코드, JSON-RPC, 툴 정의 전무 |
| **정적 콘텐츠 저장소** | ✅ | 마크다운 데이터 + 빌드 파이프라인 + CI |

구조적으로는 Hugo/Jekyll 블로그, 디자인 토큰 저장소와 같은 계열입니다 — **단일 원본에서 여러 포맷으로 배포**하는 패턴입니다.

> 이 저장소의 `CLAUDE.md`는 원본에 없던 파일로, 본 fork에서 추가한 **Claude Code 프로젝트 지침**입니다. Skill이 아닙니다.
>
> 다만 이 저장소를 **MCP 서버로 감싸는 것은 가능**합니다 (§10 참고).

### Q. API 토큰이 필요한가?

**필요 없습니다.** 저장소 전체에 API 키·토큰·시크릿·`.env` 파일이 존재하지 않습니다.

| 작업 | 토큰 필요 |
|---|---|
| 덱 다운로드 및 학습 | 불필요 |
| 로컬 `make build` | 불필요 |
| Anki 앱 사용 | 불필요 |
| fork에서 CI 실행 | 불필요 (GitHub 자동 발급) |
| AnkiWeb 동기화 | 계정만 필요 (무료, API 토큰 아님) |

워크플로의 `secrets.GITHUB_TOKEN`은 **GitHub Actions가 실행 시 자동 생성**하는 임시 토큰입니다. Nix가 GitHub API rate limit(비인증 60회/시 → 인증 5,000회/시)에 걸리지 않도록 전달하는 용도이며, 사용자가 별도로 만들 필요가 없습니다.

또한 이 프로젝트는 **LLM/AI를 전혀 사용하지 않습니다.** 순수 텍스트 변환 파이프라인입니다.

### Q. 왜 GitHub에서 인기가 있나?

1. **압도적인 콘텐츠 볼륨** — 1,000장 이상을 개인이 만들려면 수백 시간. 특히 Rust 557장은 공식 Rust Book 전체 커버 수준.
2. **Anki 커뮤니티의 규모** — 의대생·언어학습자 중심의 거대한 사용자층. 반면 양질의 "프로그래밍 Anki 덱"은 희소했음.
3. **마크다운 기반의 개발자 친화성** — Git 버전 관리, PR 기여, diff 리뷰, 원하는 에디터 사용이 모두 가능. Anki 기본 GUI 에디터의 약점을 정확히 보완.
4. **재현 가능한 빌드** — Nix flake + 해시 고정 + 매트릭스 CI + 자동 릴리스. 베스트 프랙티스 쇼케이스 성격.
5. **제작자 인지도** — ad-si (Adrian Sieber)는 TaskLite, Perspec 등 여러 오픈소스를 만든 개발자로 초기 확산에 유리.

**구조적 강점**: 단일 원본(`cards.md`) 하나만 수정하면 4가지 산출물이 자동 갱신되어 유지보수 비용이 사실상 0에 수렴합니다.

**한계 / 개선 여지** (= 기여 또는 차별화 포인트)

- Godot/Wolfram README에 복사-붙여넣기 흔적이 남아 있음 (Godot README에 Wolfram 링크가 그대로 존재)
- 일부 README에 카드 수가 `Over TODO`로 표기됨
- 덱 간 품질 편차가 큼 (Rust 557장 vs C 46장)
- **JavaScript/TypeScript, Python, Java, React 등 메이저 언어 덱이 없음**

---

## 9. 로컬 에이전트 구축 관점에서의 가치

**직접적인 에이전트 코드는 없지만, 재료로서는 최상급입니다.**

### 도움이 되지 않는 부분
에이전트 코드, LLM 호출, 툴 정의, MCP 서버가 전혀 없습니다. "에이전트 만드는 법"을 배울 요소는 없습니다.

### 도움이 되는 부분

**① 즉시 사용 가능한 고품질 Q&A 데이터셋**

`cards.md`가 이미 구조화된 질문/정답 쌍이므로 다음 용도로 바로 전환 가능합니다.

- RAG 지식베이스
- 에이전트 평가셋(eval set) ← **가장 추천**
- 파인튜닝 데이터
- 프롬프트 few-shot 예제

```python
import re

def parse_cards(path):
    raw = open(path, encoding="utf-8").read()
    body = re.sub(r"^---\n.*?\n---\n", "", raw, count=1, flags=re.S)
    qa = []
    for chunk in re.split(r"^-{3,}$", body, flags=re.M):
        chunk = chunk.strip()
        if ". . ." not in chunk:
            continue
        q, a = chunk.split(". . .", 1)
        qa.append({"question": q.strip(), "answer": a.strip()})
    return qa

cards = parse_cards("rust/cards.md")
print(len(cards), "개 파싱 완료")
```

> 활용 예: 로컬 에이전트에 Rust 질문 557개를 던져 정답률을 측정하면, 모델 교체 시 객관적 비교가 가능한 벤치마크가 됩니다.

**② 배울 만한 설계 패턴**

| 이 저장소의 패턴 | 에이전트 프로젝트 적용 |
|---|---|
| 마크다운 = 단일 원본 | 프롬프트/스킬을 마크다운으로 버전 관리 |
| `common.mk` 공유 | 에이전트별 공통 설정 재사용 |
| Nix로 환경 고정 | 실행 환경 재현성 확보 (Python/CUDA 버전 지옥 해소) |
| 매트릭스 CI | 여러 모델 동시 벤치마크 |
| `fail-fast: false` | 일부 실패해도 나머지 평가 계속 |

**③ MCP 서버로 감싸기 좋은 구조**

```
mcp-flashcards 서버가 노출할 툴 예시
  search_cards(query, language)   키워드로 카드 검색
  random_card(language, count)    랜덤 출제
  quiz_me(topic)                  대화형 퀴즈 진행
  add_card(question, answer)      새 카드 추가
  export_deck(language)           .apkg 빌드 트리거
```

이렇게 하면 "Rust 소유권 카드 5장 뽑아서 테스트해줘" 같은 자연어 명령으로 개인 학습 에이전트를 구성할 수 있습니다.

### 결론
> "이 저장소로 에이전트를 만든다"가 아니라 **"이 저장소를 에이전트에게 먹인다"** 가 정확한 포지셔닝입니다.

---

## 10. React / PHP로 재구현하기

본질이 **텍스트 파싱 + 렌더링**이므로 어떤 언어로도 구현 가능합니다.

### 10.1 React 버전 (권장)

```
cards.md (원본 그대로 재사용)
   │
   ├─ [빌드타임] Node 스크립트 파싱 → cards.json
   │
   └─ React 앱
       ├─ <FlashCard />      카드 뒤집기 애니메이션
       ├─ <DeckSelector />   언어 선택
       ├─ <SRSEngine />      SM-2 간격반복 알고리즘
       ├─ <StatsDashboard /> 학습 통계
       └─ localStorage / Supabase 진도 저장
```

**파서 (Node.js)**

```javascript
// scripts/parse-cards.mjs
import fs from "node:fs";

export function parseCards(md, deckName) {
  const body = md.replace(/^---\n[\s\S]*?\n---\n/, "");
  return body
    .split(/^-{3,}$/m)
    .map((s) => s.trim())
    .filter((s) => s.includes(". . ."))
    .map((chunk, i) => {
      const [front, back] = chunk.split(". . .");
      return {
        id: `${deckName}-${i}`,
        deck: deckName,
        front: front.trim(),
        back: back.trim(),
      };
    });
}

const decks = ["rust", "sqlite", "lua", "godot", "c", "wolfram-language"];
const all = decks.flatMap((d) =>
  parseCards(fs.readFileSync(`${d}/cards.md`, "utf8"), d)
);
fs.writeFileSync("src/data/cards.json", JSON.stringify(all, null, 2));
console.log(`${all.length}장 파싱 완료`);
```

**카드 컴포넌트**

```jsx
// src/components/FlashCard.jsx
import { useState } from "react";
import ReactMarkdown from "react-markdown";
import { Prism as SyntaxHighlighter } from "react-syntax-highlighter";

export default function FlashCard({ card, onGrade }) {
  const [flipped, setFlipped] = useState(false);

  return (
    <div className="card-container" onClick={() => setFlipped((f) => !f)}>
      <div className={`card ${flipped ? "flipped" : ""}`}>
        <div className="face front">
          <ReactMarkdown>{card.front}</ReactMarkdown>
        </div>
        <div className="face back">
          <ReactMarkdown
            components={{
              code: ({ className, children }) => (
                <SyntaxHighlighter
                  language={(className || "").replace("language-", "")}
                >
                  {String(children)}
                </SyntaxHighlighter>
              ),
            }}
          >
            {card.back}
          </ReactMarkdown>
        </div>
      </div>

      {flipped && (
        <div className="grade-buttons">
          <button onClick={() => onGrade(card.id, 0)}>다시</button>
          <button onClick={() => onGrade(card.id, 3)}>어려움</button>
          <button onClick={() => onGrade(card.id, 4)}>보통</button>
          <button onClick={() => onGrade(card.id, 5)}>쉬움</button>
        </div>
      )}
    </div>
  );
}
```

**SM-2 간격반복 알고리즘** (Anki의 핵심)

```javascript
// src/lib/sm2.js
export function sm2(card, quality) {
  let { ef = 2.5, interval = 0, reps = 0 } = card;

  if (quality < 3) {
    reps = 0;
    interval = 1;
  } else {
    reps += 1;
    if (reps === 1) interval = 1;
    else if (reps === 2) interval = 6;
    else interval = Math.round(interval * ef);
  }

  ef = Math.max(
    1.3,
    ef + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02))
  );

  return { ...card, ef, interval, reps, due: Date.now() + interval * 86400000 };
}
```

**추천 스택**

| 용도 | 선택 |
|---|---|
| 프레임워크 | Next.js (App Router) 또는 Vite + React |
| 마크다운 | `react-markdown` + `remark-gfm` |
| 코드 하이라이팅 | `shiki` 또는 `react-syntax-highlighter` |
| 애니메이션 | `framer-motion` (3D 카드 뒤집기) |
| 스타일 | Tailwind CSS |
| 상태관리 | Zustand |
| 저장 | localStorage → Supabase |
| 배포 | Vercel |

> PWA로 구성하면 모바일에 설치되어 오프라인 학습이 가능합니다. iOS Anki가 유료($25)라는 점에서 실질적인 경쟁력이 있습니다.

### 10.2 PHP 버전 (서버·회원제 중심)

```php
<?php
// app/Services/CardParser.php
class CardParser
{
    public static function parse(string $path, string $deck): array
    {
        $md = file_get_contents($path);
        $body = preg_replace('/^---\R.*?\R---\R/s', '', $md, 1);

        $chunks = preg_split('/^-{3,}$/m', $body);
        $cards = [];

        foreach ($chunks as $i => $chunk) {
            $chunk = trim($chunk);
            if ($chunk === '' || !str_contains($chunk, '. . .')) {
                continue;
            }
            [$front, $back] = explode('. . .', $chunk, 2);
            $cards[] = [
                'id'    => "{$deck}-{$i}",
                'deck'  => $deck,
                'front' => trim($front),
                'back'  => trim($back),
            ];
        }
        return $cards;
    }
}
```

**추천 스택**: Laravel 11 + Blade + Livewire / `league/commonmark` / `spatie/shiki-php` / MySQL / 아임포트·토스페이먼츠

> 회원제 + 결제 + 관리자 페이지가 중심이면 Laravel이 개발 속도에서 유리합니다. 다만 카드 뒤집기 등 인터랙션 품질은 React가 우위입니다.

### 10.3 권장 순위

1. **Next.js + Supabase + Tailwind** — 프론트/백 통합, Vercel 무료 배포, PWA 지원
2. **Vite + React (localStorage)** — MVP 최단 경로, 서버 비용 0
3. **Laravel** — 한국 결제 중심 서비스일 경우

### 10.4 로드맵 (재구현)

| 기간 | 작업 |
|---|---|
| Week 1 | 파서 + 카드 뒤집기 UI |
| Week 2 | SM-2 알고리즘 + localStorage 진도 저장 |
| Week 3 | 통계 대시보드 + 다크모드 + PWA |
| Week 4 | React/JS 오리지널 덱 작성 |
| Week 5+ | 회원가입 + 결제 |

> **핵심**: `cards.md` 포맷을 그대로 유지하면 원본 저장소 업데이트를 `git pull`만으로 반영할 수 있습니다.

---

## 11. 수익화 전략

### 11.1 라이선스 사전 확인 (필수)

```
루트에 LICENSE 파일 없음  →  기본값은 All Rights Reserved
rust/license.txt 만 존재  →  Rust 덱은 별도 조건
Rust Book 기반 카드        →  MIT / Apache-2.0 (저작자 표시 필요)
Reddit u/WebDev193 크레딧 존재
```

| 구역 | 내용 |
|---|---|
| 🟢 **안전** | 빌드 구조/파이프라인 모방 (아이디어에는 저작권 없음) / 직접 작성한 새 카드 / 콘텐츠 미포함 오픈소스 뷰어 |
| 🟡 **주의** | 원본 카드를 유료 서비스에 포함 / 원본 카드 번역본 판매 |
| 🔴 **금지** | 그대로 복사해 유료 판매 / 출처 은폐 |

> **권장 조치**: 원본 저장소에 이슈를 열어 상업적 이용 가능 여부를 먼저 확인하고, 그 대가로 신규 덱(React/JS/Python) 기여를 제안하는 것이 가장 좋은 접근입니다.

### 11.2 TIER S — 최우선

**S-1. 없는 언어 덱을 직접 작성해 판매 (라이선스 리스크 0)**

| 언어/주제 | 시장 | 난이도 | 비고 |
|---|---|---|---|
| JavaScript / TypeScript | 최대 | 쉬움 | 사용자 수 1위 |
| Python | 최대 | 쉬움 | 입문자 유입 많음 |
| React / Next.js | 큼 | 쉬움 | 본인 전문 분야 |
| Java / Spring | 큼 | 중간 | 국내 SI·대기업 수요 |
| Go | 중간 | 쉬움 | 백엔드 인기 상승 |
| PHP / Laravel | 중간 | 쉬움 | 본인 전문 분야 |
| Docker / Kubernetes | 큼 | 중간 | DevOps 수요 증가 |
| AWS 자격증 | 큼 | 중간 | 자격증 = 구매 전환 높음 |
| 코딩테스트 알고리즘 | 큼 | 중간 | 취업 준비 필수 |
| 정보처리기사 | 중간 | 쉬움 | 국내 한정 블루오션 |

판매 채널: Gumroad(수수료 10%), Lemon Squeezy(세금 자동 처리), 크몽/탈잉(국내), Udemy 번들, 자체 사이트(토스페이먼츠)

```
단품 덱 (300장)            $9 ~ $19
번들 (5개 언어)            $29 ~ $49
평생 이용권                $79 ~ $99

시뮬레이션: $19 덱 × 월 50개 = $950/월 (약 130만원)
```

**S-2. 웹 학습 플랫폼 (구독형)**

Anki의 약점(구식 UI, 덱 탐색 어려움, iOS 유료, 복잡한 설정, 개발자 특화 기능 부재)을 겨냥합니다.

- 브라우저 즉시 시작 (설치 불필요)
- 다크모드 + 코드 하이라이팅
- PWA → 모바일 설치, 오프라인 지원
- 학습 통계 대시보드 + 스트릭
- 리더보드/배지 (게이미피케이션)
- AI 튜터 (이해 안 되는 카드 추가 설명)
- 면접 모드 (타이머 + 음성 답변 연습)

```
Free       하루 20장, 덱 2개
Pro        $5/월 또는 $39/년 — 무제한 + AI 설명 + 통계 + 오프라인
Lifetime   $99
Team       $3/인/월 (사내 온보딩)

MAU 10,000 × 전환율 3% = 300명 × $5 = $1,500/월 (약 200만원)
MAU 50,000 × 전환율 4% = 2,000명 × $5 = $10,000/월 (약 1,300만원)
```

**S-3. 기업 온보딩 / 사내 교육 (B2B)**

```
사내 위키 / README / 코딩 컨벤션
      ↓ (LLM 파이프라인)
"우리 회사 API 인증 방식은?" 같은 카드 자동 생성
      ↓
신입 개발자가 Anki로 학습 → 온보딩 3주 → 1주로 단축
```

```
스타트업 (~50명)     월 50만 ~ 100만원
중견 (~500명)        월 300만 ~ 500만원
엔터프라이즈          연 3,000만원 ~ (온프레미스 + 커스텀)
```

세일즈 논리: *"온보딩 2주 단축 = 1인당 약 400만원 절감. 연 10명 채용 시 4,000만원 절감 대비 연 1,200만원 비용."*

### 11.3 TIER A — 부수입

**A-1. 콘텐츠 마케팅 → 광고/제휴**
유튜브("Rust 카드 100장으로 1주일 만에 배우기"), 블로그 SEO, X 자동 포스팅 봇.
월 방문 10만 기준 광고 $500~1,500 + 제휴 $500~2,000. **다른 티어의 트래픽 엔진** 역할이 본질적 가치.

**A-2. 덱 빌더 SaaS**
누구나 마크다운으로 덱을 만들고 판매하는 플랫폼. 이 저장소의 빌드 파이프라인을 그대로 SaaS화.
`Creator Free`(덱 3개) / `Creator Pro` $12/월 / 마켓 수수료 15%.

**A-3. AI 카드 자동 생성기**
PDF·URL·GitHub 레포·유튜브 영상 → 카드 50~200장 자동 생성.
크레딧제(100장 $5 / 500장 $20 / 무제한 $25월). LLM API 원가 관리(배치 처리, 프롬프트 캐싱) 필수.

### 11.4 TIER B — 소규모

- **물리 카드 굿즈** — Printful/Printify POD, 재고 0. 판매가 $29~39, 마진 약 $15.
- **전자책 + 덱 번들** — PDF는 기존 파이프라인이 이미 생성하므로 추가 작업 없음. Gumroad $24 / 국내 전자책 플랫폼.
- **GitHub Sponsors** — $3 이름 등재 / $10 신규 덱 조기 접근 / $50 덱 요청권 / $500 기업 로고. 수익보다 신뢰도·커뮤니티 확보가 목적.

### 11.5 실행 로드맵 (수익화)

| 시점 | 목표 |
|---|---|
| Month 1 | React 덱 200장 무료 공개 → GitHub ⭐300, 이메일 500 |
| Month 2 | Next.js 웹 뷰어 MVP 배포 → MAU 3,000 |
| Month 3 | 유료 덱 2종 출시 (Gumroad + 크몽) → 첫 $500 |
| Month 4 | 구독 오픈 ($5/월) + AI 튜터 → 유료 100명, MRR $500 |
| Month 6 | B2B 영업 시작 (지인 회사 무료 → 레퍼런스) → MRR $2,000 |
| Month 12 | B2B 2~3건 + 구독 500명 → MRR $10,000 (약 1,300만원) |

### 11.6 우선순위 결론

1. **React/JS 오리지널 덱 작성** — 라이선스 프리, 전문 분야, 시장 최대. 무료 배포로 인지도 선확보.
2. **Next.js 웹 뷰어** — 2주 내 MVP, Vercel 무료로 비용 0.
3. **B2B 온보딩 솔루션** — 객단가 최고, 소속 회사부터 레퍼런스 확보.

> 플래시카드는 직접 써봐야 가치를 체감하는 제품입니다. 무료 200장으로 효과를 경험시킨 뒤 유료 전환을 유도하는 **프리미엄(Freemium) 전략**이 핵심입니다.

---

## 12. 핵심 요약

| 질문 | 답 |
|---|---|
| 이게 뭐야? | 마크다운 플래시카드 1,035장 + Anki/PDF 자동 빌드 파이프라인 |
| 플러그인/Skill/MCP? | 전부 아님. 정적 콘텐츠 저장소 |
| API 토큰 필요? | 불필요. `secrets.GITHUB_TOKEN`은 Actions 자동 발급 |
| 왜 유명해? | 압도적 볼륨 + Anki 커뮤니티 + 마크다운 기반 + Nix 재현성 + 제작자 인지도 |
| 로컬 에이전트에 도움? | 직접적으론 아니지만 **eval 데이터셋**으로는 최상급 |
| React/PHP 가능? | 매우 쉬움. 정규식 `^-{3,}$` + `. . .` 분할이면 파서 완성 |
| 수익화? | ① 없는 언어 덱 판매 ② 웹 구독 서비스 ③ B2B 온보딩 |
| 시작은 뭐부터? | **React 덱 200장 무료 공개** |

---

*이 문서는 저장소 전체 파일을 직접 읽고 분석해 작성했습니다.*
