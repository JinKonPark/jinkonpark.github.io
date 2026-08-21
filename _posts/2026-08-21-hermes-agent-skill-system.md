---
title: "스킬 199개를 시스템 프롬프트에 넣지 않고 관리하는 방법"
date: 2026-08-21 13:40:00 +0900
categories: [AI Agents, Hermes Agent]
tags: [llm, agent, skills, prompt-cache]
---

<style>
.dg { --core:#b4530a; --edge:#2f6f8f; --warn:#8a6d00; --danger:#a32020; --ok:#2f7d32;
      --fill-core:#fdf0e4; --fill-edge:#e9f2f6; --fill-warn:#fdf6e0;
      --fill-danger:#fbeaea; --fill-ok:#eaf4ea;
      --line:#e3e0d9; --muted:#6b6862; }
html[data-mode="dark"] .dg {
      --core:#f0954a; --edge:#7fbcd8; --warn:#e0be4c; --danger:#e88b8b; --ok:#7ec482;
      --fill-core:#2e2118; --fill-edge:#18262d; --fill-warn:#2c2718;
      --fill-danger:#2e1c1c; --fill-ok:#1a2a1c;
      --line:#33313a; --muted:#9d9891; }
.dg { margin: 1.5rem 0; }
.dg svg { display:block; width:100%; height:auto; }
.dg figcaption { margin-top:.6rem; font-size:.88rem; color:var(--muted); line-height:1.6; }
</style>
에이전트에 붙일 스킬이 200개 가까이 되면 설명을 전부 시스템 프롬프트에 넣을 수 없다. `NousResearch/hermes-agent`는 스킬 199개를 관리하면서 프롬프트 예산을 15KB 안쪽으로 유지한다. 어떻게 하는지 코드를 따라가 봤다.

## 문제

스킬 하나는 폴더 하나이고, 필수 파일은 `SKILL.md` 한 장이다. 이 저장소에는 기본 활성 82개와 선택 설치 117개, 합쳐서 199개가 있다.

문제는 명확하다. 스킬 하나의 설명을 200자씩만 잡아도 199개면 40KB다. 시스템 프롬프트 앞부분에 그만한 분량이 들어가면 프롬프트 캐시가 감당하지 못한다. 그렇다고 스킬 목록을 아예 빼면 에이전트가 무엇이 있는지 모른다.

## 처음 시도한 접근

스킬 설명을 짧게 쓰자고 규약으로 정하는 방법이 먼저 떠올랐다. 그런데 규약만으로는 지켜지지 않는다. 설명이 길수록 스킬이 잘 선택되니 작성자로서는 길게 쓸 이유가 있다.

이 저장소는 규약 대신 **상수 하나**로 이 문제를 잡는다.

```python
SKILL_PROMPT_DESC_LIMIT = 60
```

시스템 프롬프트에는 스킬 이름과 설명 60자만 들어간다. 스킬당 약 70~80바이트이고, 199개를 다 합쳐도 15KB를 넘지 않는다.

## 바꾼 설계

### 점진적 공개 3단계

전체 내용은 세 단계로 나뉜다.

| 단계 | 로드 시점 | 내용 |
| --- | --- | --- |
| 1 | 항상 | 이름 + 설명 60자 |
| 2 | 스킬이 선택됐을 때 | `SKILL.md` 본문 |
| 3 | 필요할 때만 | `references/`, `scripts/` 등 부속 파일 |

부속 폴더는 네 종류만 쓸 수 있다 — `references/`(88개 스킬), `scripts/`(53), `templates/`(21), `assets/`(2). 절반 넘는 스킬은 `SKILL.md` 한 장으로 끝난다. 본문 상한은 100,000자이고, 넘으면 참조 파일로 쪼개라는 신호다.

여기서 눈여겨볼 점은 상한값 자체가 아니라 **어느 단계에 무엇을 둘지 강제하는 구조**다. 1단계를 60자로 고정하면 나머지 설계가 따라온다.

<figure class="dg">
<svg viewBox="0 0 880 290" role="img" aria-label="점진적 공개 3단계와 각 단계의 프롬프트 비용">
<defs>
  <marker id="ss1-a" markerWidth="9" markerHeight="9" refX="8" refY="3.2" orient="auto">
    <path d="M0,0 L8,3.2 L0,6.4 z" fill="currentColor"/>
  </marker>
</defs>
<rect x="20" y="30" width="270" height="100" rx="11"
      fill="var(--fill-core)" stroke="var(--core)" stroke-width="2"/>
<text x="40" y="56" font-size="13.5" font-weight="700" fill="var(--core)">1단계 · 항상 로드</text>
<text x="40" y="80" font-size="12.5" fill="var(--core)">이름 + 설명 60자</text>
<text x="40" y="102" font-size="12.5" fill="var(--muted)">스킬당 70~80바이트</text>
<text x="40" y="122" font-size="12.5" font-weight="700" fill="var(--core)">199개 전부 = 15KB 이내</text>

<g color="var(--edge)">
  <path d="M296,80 L336,80" stroke="currentColor" stroke-width="2.2" marker-end="url(#ss1-a)"/>
</g>
<text x="316" y="70" font-size="11" text-anchor="middle" fill="var(--edge)">선택됨</text>

<rect x="342" y="30" width="250" height="100" rx="11"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<text x="362" y="56" font-size="13.5" font-weight="700" fill="var(--edge)">2단계 · 선택 시</text>
<text x="362" y="80" font-size="12.5" fill="var(--edge)">SKILL.md 본문</text>
<text x="362" y="102" font-size="12.5" fill="var(--muted)">상한 100,000자</text>

<g color="var(--edge)">
  <path d="M598,80 L638,80" stroke="currentColor" stroke-width="2.2" marker-end="url(#ss1-a)"/>
</g>
<text x="618" y="70" font-size="11" text-anchor="middle" fill="var(--edge)">필요 시</text>

<rect x="644" y="30" width="216" height="100" rx="11"
      fill="none" stroke="var(--line)" stroke-width="1.5"/>
<text x="664" y="56" font-size="13.5" font-weight="700" fill="var(--muted)">3단계 · 필요할 때</text>
<text x="664" y="80" font-size="12.5" fill="var(--muted)">references/ scripts/</text>
<text x="664" y="102" font-size="12.5" fill="var(--muted)">templates/ assets/</text>

<rect x="20" y="160" width="840" height="110" rx="11"
      fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.6"/>
<text x="40" y="188" font-size="13.5" font-weight="700" fill="var(--warn)">SKILL_PROMPT_DESC_LIMIT = 60 — 같은 상수, 네 곳이 다르게 반응한다</text>
<text x="40" y="216" font-size="12.5" fill="var(--warn)">렌더링: 조용히 잘라낸다 desc[:57] + "…"</text>
<text x="40" y="238" font-size="12.5" fill="var(--warn)">린터: 경고만 낸다</text>
<text x="450" y="216" font-size="12.5" fill="var(--warn)">하드 검증기: 신규 생성일 때만 거부</text>
<text x="450" y="238" font-size="12.5" fill="var(--danger)">CI: 실패시킨다</text>
<text x="40" y="260" font-size="12" fill="var(--muted)">기존 스킬을 고칠 때는 검증을 일부러 건너뛴다 — 전부 막는 대신 신규만 막는다</text>
</svg>
<figcaption>그림 1. 1단계를 60자로 고정하면 나머지 설계가 따라온다. 눈여겨볼 점은 상한값이 아니라 <strong>어느 단계에 무엇을 둘지 강제하는 구조</strong>다.</figcaption>
</figure>


### 같은 상수를 네 곳이 다르게 다룬다

흥미로운 대목이 있다. 60자 제한을 검사하는 곳은 네 군데인데, 걸렸을 때 반응이 전부 다르다.

- 렌더링: 조용히 잘라낸다 (`desc[:57] + "..."`)
- 린터: 경고만 낸다
- 하드 검증기: **신규 생성일 때만** 거부한다
- CI: 실패시킨다

기존 스킬을 고칠 때는 검증을 일부러 건너뛴다. 이미 60자를 넘긴 스킬의 다른 부분을 손보려다 막히는 상황을 피하려는 선택이다. 규칙을 새로 도입할 때 기존 자산을 어떻게 다룰지에 대한 답이기도 하다. 전부 막는 대신 신규만 막는다.

### 캐시를 지키는 두 가지 조치

프롬프트 캐시를 살리려고 두 가지를 한다.

첫째, 스킬 목록을 시스템 프롬프트의 **변동 구간 맨 앞**에 둔다. 스킬이 추가돼도 그 앞쪽은 그대로 남는다.

둘째, 슬래시 커맨드로 스킬을 부를 때 본문을 시스템 프롬프트가 아니라 **사용자 메시지로** 넣는다. 시스템 프롬프트를 건드리지 않으니 캐시도 깨지지 않는다.

### 탐색은 4단 우선순위

같은 이름의 스킬이 여러 곳에 있으면 먼저 찾은 쪽이 이긴다.

```
프로젝트 → 프로필(~/.hermes/skills/) → 외부 디렉터리 → 조직 미러
```

프로젝트가 1순위라서 저장소마다 같은 이름 스킬을 덮어쓸 수 있다. 그런데 1순위인 프로젝트 스킬에만 격리 절차가 하나 더 붙는다. 이유는 코드 주석에 정확히 적혀 있다 — 신뢰는 저장소 단위로 한 번 주는데, 내용은 `pull`할 때마다 바뀌기 때문이다.

스캐너가 죽으면 통과가 아니라 차단이다. fail-closed다.

### 설치 정책은 신뢰도 × 판정 매트릭스

외부 스킬을 설치할 때는 출처 신뢰도와 스캔 판정을 교차해서 결정한다.

> **이 가드가 막는 것.** 강제 플래그로도 위험 판정은 뚫지 못한다. `--force` 가 있어도 critical 이 하나라도 잡히면 설치는 거부된다.
{: .prompt-danger }

판정을 정하는 규칙도 명확하다. critical이 하나라도 있으면 위험, high가 있으면 주의, medium과 low만 있으면 안전으로 친다. 낮은 등급 발견은 참고 정보로만 다룬다.

신뢰 저장소는 네 곳으로 못 박혀 있고, 접두사만 같은 형제 저장소는 일부러 신뢰하지 않는다. `anthropics/`를 신뢰한다고 해서 이름이 비슷한 다른 계정까지 믿지는 않는다는 뜻이다.

### 에이전트가 만든 스킬은 스캔이 꺼져 있다

가장 솔직한 주석이 여기 있다. 에이전트가 직접 만든 스킬은 보안 스캔이 기본으로 꺼진다. 에이전트는 이미 터미널 도구로 같은 코드를 아무 게이트 없이 실행할 수 있기 때문이다. 문서 작성만 검사해 봐야 더 안전해지지는 않고 번거로움만 늘어난다.

> **이 문서가 정정하는 통념 하나.** 에이전트가 직접 만든 스킬은 보안 스캔이 기본으로 꺼져 있다. 에이전트는 이미 터미널 도구로 같은 코드를 아무 게이트 없이 실행할 수 있으므로, 문서 작성만 검사해 봐야 더 안전해지지 않는다.
{: .prompt-info }

스캔은 "밖에서 들어오는 것"을 막는 장치라는 뜻이다. 위협 모델을 분명히 정하고 거기에 맞춰 검사를 배치한 사례로 읽힌다.

## 검증

2026-08 시점의 main 브랜치를 클론해 파일 단위로 하나씩 확인했다.

| 구분 | 항목 |
| --- | --- |
| **확인함** | 스킬 199개(82 + 117) · 부속 폴더 4종을 쓰는 스킬 개수 · 60자 상수와 참조 네 지점 · 탐색 우선순위 · 설치 정책 매트릭스 |
| **확인하지 못함** | 15KB 프롬프트 예산이 실제 요청에서 유지되는지 — 계산으로 어림했을 뿐 요청을 캡처하지 않았다 |

CI가 막는 항목에는 스타일 규칙도 있다. 설명이 마침표로 끝나면 안 되고, 홍보성 단어(powerful, comprehensive, seamless 등)를 쓰면 실패한다. 예외 목록은 지금 비어 있다 — 199개 전부가 규칙을 지키고 있다는 뜻이다.

## 한계

서명 검증은 어디에도 없다. 내용으로 SHA-256을 계산하기는 하지만 캐시 키와 갱신 감지에만 쓴다. 처음 본 것을 일단 신뢰하는 방식(TOFU)이고, 스캔도 휴리스틱일 뿐 암호학적 보증은 아니다.

플러그인에 딸려 오는 스킬은 목록에 뜨지 않고 이름을 직접 적어야만 로드된다. 60자 예산을 지키려는 선택인데, 대신 에이전트가 그런 스킬을 스스로 찾아내지 못한다. 이 맞바꿈이 실사용에서 어느 쪽으로 기우는지는 확인하지 못했다.

이름 때문에 오해하기 쉬운 모듈이 여럿이라는 점도 적어 둔다. 출처 추적처럼 보이는 모듈이 알고 보면 전경 작업과 백그라운드 작업을 구분하는 78줄짜리 컨텍스트 변수이고, 원장처럼 보이는 모듈은 스스로 "게이트가 아니라 계측"이라고 밝힌다. 이름만 보고 역할을 짐작하면 틀린다.
