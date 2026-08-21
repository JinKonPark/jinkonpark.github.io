---
title: "정본 문서가 있어도 온보딩이 막히는 이유"
date: 2026-08-21 14:00:00 +0900
categories: [AI Agents, Hermes Agent]
tags: [llm, agent, onboarding, testing]
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
`NousResearch/hermes-agent`에는 1,100줄짜리 정본 문서가 있다. 그런데도 코드를 고치러 들어간 첫날 막혔다. 무엇이 부족했는지, 그리고 문서와 실제가 갈리는 지점을 어떻게 찾았는지 정리한다.

## 문제

정본 문서는 임포트 의존 사슬을 정확히 적어 두었다. 어떤 모듈이 무엇을 가져오는지는 다 나와 있다.

빠진 것은 **읽는 순서**였다. 의존 그래프는 순서가 아니다. 어디가 입구인지, 무엇을 먼저 이해해야 다음이 읽히는지는 그래프만 봐서는 알 수 없다.

문서가 부실해서 생긴 문제가 아니라, 문서가 답하지 않는 질문이 따로 있다는 문제였다.

<figure class="dg">
<svg viewBox="0 0 800 250" role="img" aria-label="의존 그래프는 있지만 읽는 순서가 없다">
<defs>
  <marker id="ob1-a" markerWidth="9" markerHeight="9" refX="8" refY="3.2" orient="auto">
    <path d="M0,0 L8,3.2 L0,6.4 z" fill="currentColor"/>
  </marker>
</defs>
<rect x="16" y="30" width="360" height="200" rx="11"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<text x="36" y="56" font-size="13.5" font-weight="700" fill="var(--edge)">정본 문서가 답하는 것</text>
<text x="36" y="80" font-size="12.5" fill="var(--edge)">임포트 의존 사슬 — 무엇이 무엇을 가져오는가</text>
<circle cx="90" cy="130" r="17" fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<circle cx="190" cy="112" r="17" fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<circle cx="190" cy="170" r="17" fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<circle cx="290" cy="140" r="17" fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<g color="var(--edge)">
  <path d="M108,124 L172,114" stroke="currentColor" stroke-width="1.8" marker-end="url(#ob1-a)"/>
  <path d="M108,138 L172,166" stroke="currentColor" stroke-width="1.8" marker-end="url(#ob1-a)"/>
  <path d="M208,118 L272,134" stroke="currentColor" stroke-width="1.8" marker-end="url(#ob1-a)"/>
  <path d="M208,164 L272,146" stroke="currentColor" stroke-width="1.8" marker-end="url(#ob1-a)"/>
</g>
<text x="36" y="212" font-size="12" fill="var(--muted)">1,100줄 · 정확하다</text>

<rect x="424" y="30" width="360" height="200" rx="11"
      fill="var(--fill-danger)" stroke="var(--danger)" stroke-width="1.6"
      stroke-dasharray="6 5"/>
<text x="444" y="56" font-size="13.5" font-weight="700" fill="var(--danger)">답하지 않는 것</text>
<text x="444" y="80" font-size="12.5" fill="var(--danger)">읽는 순서 — 어디가 입구인가</text>
<rect x="444" y="102" width="70" height="34" rx="8" fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.5"/>
<text x="479" y="124" font-size="13" text-anchor="middle" fill="var(--warn)">?</text>
<g color="var(--muted)"><path d="M520,119 L556,119" stroke="currentColor" stroke-width="1.8" marker-end="url(#ob1-a)"/></g>
<rect x="562" y="102" width="70" height="34" rx="8" fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.5"/>
<text x="597" y="124" font-size="13" text-anchor="middle" fill="var(--warn)">?</text>
<g color="var(--muted)"><path d="M638,119 L674,119" stroke="currentColor" stroke-width="1.8" marker-end="url(#ob1-a)"/></g>
<rect x="680" y="102" width="70" height="34" rx="8" fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.5"/>
<text x="715" y="124" font-size="13" text-anchor="middle" fill="var(--warn)">?</text>
<text x="444" y="170" font-size="12.5" fill="var(--danger)">의존 그래프는 순서가 아니다</text>
<text x="444" y="192" font-size="12.5" fill="var(--danger)">문서와 코드가 갈리는 지점 11군데</text>
</svg>
<figcaption>그림 1. 문서가 부실해서 생긴 문제가 아니라, <strong>문서가 답하지 않는 질문</strong>이 따로 있다는 문제였다.</figcaption>
</figure>


## 처음 시도한 접근

문서를 그대로 요약해서 가이드를 만들려고 했다. 두 가지 이유로 잘 되지 않았다.

첫째, 요약해도 순서 문제는 그대로 남는다. 원본에 없는 정보는 요약에도 없다.

둘째, 요약하다 보면 문서와 코드가 어긋난 지점까지 그대로 옮기게 된다. 실제로 확인해 보니 어긋난 곳이 11군데였다.

그래서 문서를 옮기는 대신 **코드를 직접 확인해서** 다시 쓰기로 했다.

## 바꾼 설계

### 문서 교정을 별도 항목으로 표시했다

문서와 코드가 갈리는 지점을 발견할 때마다 본문에 "문서 교정"으로 표시했다. 말없이 고쳐 두면 다음 사람이 원본 문서를 볼 때 또 헷갈린다. 어긋났다는 사실 자체가 정보다.

가장 큰 모순은 테스트 실행 방법이었다.

- PR 템플릿: `pytest tests/ -q`를 체크리스트로 요구
- 개발 가이드: "맨 pytest 절대 금지, 반드시 래퍼 스크립트"

> **이 문서가 정정하는 통념 하나.** PR 템플릿은 `pytest tests/ -q` 를 체크리스트로 요구하고, 개발 가이드는 "맨 pytest 절대 금지, 반드시 래퍼 스크립트"라고 적는다. **정답은 래퍼다.** 문서에만 있고 실제로는 없는 플래그와 파일도 있어서, 적힌 대로 실행하면 그냥 실패한다.
{: .prompt-danger }

### 실습을 두 개 넣었다

읽기만 해서는 구조가 손에 안 붙는다. 직접 따라 해 보는 실습을 두 개 넣었다.

1. 도구 추가 — `word_count` 같은 작은 도구를 처음부터 만든다
2. 플러그인과 스킬 작성

표준 예제로는 저장소에 있는 71줄짜리 도구를 골랐다. 전체를 읽고 그대로 따라 쓰기 좋은 크기다. 예제가 너무 크면 어디까지가 따라 할 부분인지 감이 안 온다.

### 가장 자주 걸리는 함정

도구를 만들 때는 거의 모두가 같은 곳에서 막힌다. 레지스트리에 등록만 해서는 도구가 보이지 않는다. 세 관문을 다 통과해야 한다.

```
registry.register()   →   toolsets 등재   →   check_fn 런타임 게이트
```

> **핵심 함정.** `requires_env` 는 이름과 달리 **진짜 게이트가 아니다** — 표시용 메타데이터일 뿐이다. 이 필드에 환경변수를 적어 두면 게이트가 걸리려니 하고 넘어갔다간 낭패를 본다.
{: .prompt-warning }

이름이 동작을 잘못 설명하는 필드는 문서보다 코드를 봐야 알 수 있다.

### 깨면 안 되는 것 다섯 개

기여할 때 지켜야 하는 불변식을 다섯 개로 추렸다.

| 불변식 | 이유 |
| --- | --- |
| 프롬프트 캐시 불가침 | 앞쪽 컨텍스트를 건드리면 비용이 다시 든다 |
| 프로파일 안전 경로 사용 | 홈 디렉터리를 직접 조립하지 않는다 |
| 크로스플랫폼 | 경로 구분자와 셸 가정을 넣지 않는다 |
| 의존성 버전 고정 | 범위 지정 대신 정확한 버전으로 |
| 변경 감지 테스트 금지 | 개수나 목록을 고정하는 테스트를 만들지 않는다 |

마지막 항목이 특히 눈여겨볼 만하다. "스킬이 199개인지 확인" 같은 테스트를 금지한다. 이런 테스트는 기능이 깨졌을 때가 아니라 **정상적으로 늘어났을 때** 실패한다. 신호가 아니라 소음이 된다.

## 검증

이 가이드는 2026-07-29 시점 v0.19.0 스냅숏을 클론해 확인하며 썼다. 저장소 문서를 옮긴 것이 아니라 각 주장을 코드에서 다시 확인했다.

| 구분 | 항목 |
| --- | --- |
| **확인함** | 세 관문의 실제 코드 경로 · `requires_env` 가 게이트가 아니라는 점 · 문서와 실제가 갈리는 11개 지점 · 테스트 래퍼가 정답이라는 점 |
| **확인하지 못함** | 이 순서로 읽으면 온보딩이 빨라지는지 — 내 경험 하나뿐이고 다른 사람에게 시켜 보지 않았다 |

## 한계

줄 번호는 스냅숏 기준이다. 하루에 커밋이 188개쯤 들어오는 저장소라 지금은 대부분 어긋났을 것이다. 심볼 이름으로 다시 찾는 편이 안전하다.

문서 교정 11건은 그 시점의 상태다. 이후 문서가 고쳐졌을 수도 있고, 새로 어긋난 곳이 생겼을 수도 있다. 교정 목록 자체보다 **문서와 코드를 대조하며 읽는 방식**이 남길 만하다고 본다.

읽는 순서를 정본 문서에 넣으면 되지 않느냐고 반문할 수도 있다. 다만 순서는 목적에 따라 달라진다. 도구를 추가하려는 사람과 프로바이더를 붙이려는 사람은 입구가 다르다. 정본 순서를 하나로 정하는 게 옳은지는 아직 판단이 서지 않는다.
