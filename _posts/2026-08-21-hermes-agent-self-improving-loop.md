---
title: "자가개선 에이전트는 실제로 무엇을 하는가"
date: 2026-08-21 13:50:00 +0900
categories: [AI Agents, Hermes Agent]
tags: [llm, agent, self-improving, prompt-cache]
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
"스스로 개선하는 에이전트"라는 표현은 가중치가 갱신된다는 뜻으로 읽히기 쉽다. `NousResearch/hermes-agent`가 그렇게 소개하길래 코드 수준에서 실제로 무슨 일이 벌어지는지 끝까지 따라가 봤다. 결론부터 말하면 가중치는 그대로고, 전부 파일 읽기와 쓰기로 설명된다.

## 문제

에이전트가 세션에서 배운 내용을 다음 세션에 넘기려면 어딘가에 써 둬야 한다. 가장 단순한 방법은 대화 중에 "이걸 기억해 둬"라고 주입하는 방식이다.

그런데 이 방식은 프롬프트 캐시를 깬다. 대화 앞부분이 바뀌면 캐시된 접두사가 무효화되고 비용이 다시 든다. 배우는 행위 자체가 비싸지는 구조다.

## 처음 시도한 접근

이 저장소에는 "nudge"라는 이름의 메커니즘이 있다. 이름만 보면 대화에 무언가를 밀어 넣는 것처럼 읽힌다. 나도 그렇게 읽고 주입 지점부터 찾았다.

주입은 없었다. 코드 주석 두 군데가 `no nudge injection`이라고 못 박고 있다. 카운터는 두 개인데 하는 일은 하나뿐이다.

- `_turns_since_memory` — 턴 단위로 센다
- `_iters_since_skill` — 도구 반복 단위로 센다

두 카운터는 **포크를 띄울지 말지**만 결정한다. 대화에는 아무것도 들어가지 않는다. 이름이 동작을 잘못 설명하고 있는 셈이다.

<figure class="dg">
<svg viewBox="0 0 880 300" role="img" aria-label="카운터가 대화가 아니라 포크 실행만 결정하는 구조">
<defs>
  <marker id="sil1-a" markerWidth="9" markerHeight="9" refX="8" refY="3.2" orient="auto">
    <path d="M0,0 L8,3.2 L0,6.4 z" fill="currentColor"/>
  </marker>
</defs>
<rect x="20" y="26" width="250" height="120" rx="11"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<text x="40" y="52" font-size="13.5" font-weight="700" fill="var(--edge)">카운터 2개</text>
<text x="40" y="80" font-size="12.5" fill="var(--edge)">_turns_since_memory</text>
<text x="40" y="104" font-size="12.5" fill="var(--edge)">_iters_since_skill</text>
<text x="40" y="130" font-size="12" fill="var(--muted)">턴 · 도구 반복을 센다</text>

<g color="var(--core)">
  <path d="M276,86 L346,86" stroke="currentColor" stroke-width="2.2" marker-end="url(#sil1-a)"/>
</g>
<text x="311" y="76" font-size="11.5" text-anchor="middle" fill="var(--core)">임계값</text>

<rect x="352" y="26" width="250" height="120" rx="11"
      fill="var(--fill-core)" stroke="var(--core)" stroke-width="1.6"/>
<text x="372" y="52" font-size="13.5" font-weight="700" fill="var(--core)">리뷰 포크 실행</text>
<text x="372" y="80" font-size="12.5" fill="var(--core)">백그라운드 데몬 스레드</text>
<text x="372" y="104" font-size="12.5" fill="var(--core)">별도 에이전트 인스턴스</text>
<text x="372" y="130" font-size="12" fill="var(--muted)">결과는 파일로 쓴다</text>

<rect x="20" y="196" width="582" height="76" rx="11"
      fill="var(--fill-danger)" stroke="var(--danger)" stroke-width="1.6"
      stroke-dasharray="6 5"/>
<text x="40" y="224" font-size="13.5" font-weight="700" fill="var(--danger)">대화(시스템 프롬프트) — 아무것도 들어가지 않는다</text>
<text x="40" y="250" font-size="12.5" fill="var(--danger)">no nudge injection · 프롬프트 캐시가 깨지지 않는다</text>

<g color="var(--danger)">
  <path d="M477,150 L477,192" stroke="currentColor" stroke-width="2"
        stroke-dasharray="5 4"/>
  <path d="M462,168 L492,182 M492,168 L462,182" stroke="currentColor" stroke-width="2.4"/>
</g>

<rect x="636" y="26" width="224" height="246" rx="11"
      fill="none" stroke="var(--line)" stroke-width="1.4"/>
<text x="656" y="52" font-size="13" font-weight="700" fill="var(--muted)">다음 세션</text>
<text x="656" y="80" font-size="12.5" fill="var(--muted)">파일에서 읽어</text>
<text x="656" y="102" font-size="12.5" fill="var(--muted)">시스템 프롬프트에</text>
<text x="656" y="124" font-size="12.5" fill="var(--muted)">반영된다</text>
<g color="var(--core)">
  <path d="M608,86 L630,86" stroke="currentColor" stroke-width="2.2" marker-end="url(#sil1-a)"/>
</g>
</svg>
<figcaption>그림 1. 이름은 "nudge"지만 대화에는 아무것도 주입되지 않는다. 카운터는 <strong>포크를 띄울지</strong>만 결정하고, 학습 결과는 파일을 거쳐 다음 세션에 도달한다.</figcaption>
</figure>


## 바꾼 설계

### 리뷰는 인라인이 아니라 포크다

임계값에 닿으면 백그라운드 데몬 스레드에서 별도 에이전트 인스턴스가 돈다. 이유는 하나, 프롬프트 캐시다.

저장소 문서는 "세션별 프롬프트 캐싱은 불가침"이라고 선을 긋는다. 리뷰 모듈 주석에는 이 선택으로 비용이 약 26% 줄었다고 적혀 있고, 관련 이슈와 PR 번호까지 남아 있다.

### 캐시 패리티를 위해 도구 목록을 부모와 똑같이 보낸다

포크가 부모와 다른 도구 목록을 보내면 캐시 키가 달라진다. 그래서 도구 배열을 **바이트 단위까지 부모와 동일하게** 보낸다.

실제 도구 제한은 별도로 건다. 스레드 로컬 화이트리스트가 실행 시점에 막는다. 캐시 키와 실행 권한을 갈라 놓은 2층 구조다.

다른 시스템에도 그대로 가져다 쓸 만한 판단이다. 캐시 키를 결정하는 값과 권한을 결정하는 값이 같은 필드일 이유는 없다.

### 큐레이터 탈취 버그

포크가 사용자 세션을 오염시킨 사고의 흔적이 주석에 남아 있다. 특정 플래그가 없으면 포크가 실행한 하네스 턴이 사용자의 진짜 세션에 남아 **상주 명령**이 된다.

포크와 부모가 같은 저장소를 공유하면서 쓰기 경로를 격리하지 않으면 생기는 문제다. 읽기만 공유하고 쓰기는 분리해야 한다는 교훈을 사고로 얻은 것으로 보인다.

### 가장 잘 다듬어진 산출물은 금지 목록이다

리뷰가 만들어 내는 산출물 가운데 가장 정교한 쪽은 "배우지 말아야 할 것" 목록이다. 그중에서도 특히 눈에 띄는 항목이 하나 있다.

> **이 가드가 막는 것.** 부정 주장을 기록하지 말 것 — "브라우저 도구는 안 된다" 같은 문장이 한 번 굳으면 에이전트가 스스로 시도를 접고, 고쳐졌다는 증거도 영영 쌓이지 않는다.
{: .prompt-danger }

"브라우저 도구는 안 된다" 같은 문장이 한 번 굳으면, 에이전트는 몇 달이 지나도 그 문장을 근거로 스스로 시도를 접는다. 시도를 안 하니 고쳐졌다는 증거도 영영 쌓이지 않는다. 스스로 닫히는 고리다.

규칙은 한 줄로 정리돼 있다 — **장애물이 아니라 통과 방법을 적어라.**

에이전트에 메모리를 붙이는 시스템이라면 어디서든 부딪히는 문제다. 실패 경험을 그대로 저장하면 그 실패가 영영 굳어 버린다.

### 두 번째 루프가 따로 있다

리뷰 포크는 한 세션만 본다. 그래서 구조상 항목이 계속 늘어나기만 하는 편향이 생긴다.

이를 보정하려고 큐레이터가 따로 돈다. 7일마다, 그리고 2시간 이상 유휴 상태일 때 트리거된다. 크론이 아니라 **비활동**으로 트리거된다는 점이 특이하다. 큐레이터는 전체 라이브러리를 훑어 합친다.

수명 주기는 `active → stale(30일) → archived(90일)`이고, **자동 삭제는 없다.**

<figure class="dg">
<svg viewBox="0 0 880 240" role="img" aria-label="리뷰 포크와 큐레이터 두 루프의 역할 분담">
<defs>
  <marker id="sil2-a" markerWidth="9" markerHeight="9" refX="8" refY="3.2" orient="auto">
    <path d="M0,0 L8,3.2 L0,6.4 z" fill="currentColor"/>
  </marker>
</defs>
<rect x="20" y="24" width="330" height="92" rx="11"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.6"/>
<text x="40" y="50" font-size="13.5" font-weight="700" fill="var(--edge)">루프 1 · 리뷰 포크</text>
<text x="40" y="76" font-size="12.5" fill="var(--edge)">한 세션만 본다 → 항목이 계속 늘어난다</text>
<text x="40" y="100" font-size="12.5" fill="var(--muted)">트리거: 카운터 임계값</text>

<rect x="20" y="140" width="330" height="80" rx="11"
      fill="var(--fill-core)" stroke="var(--core)" stroke-width="1.6"/>
<text x="40" y="166" font-size="13.5" font-weight="700" fill="var(--core)">루프 2 · 큐레이터</text>
<text x="40" y="192" font-size="12.5" fill="var(--core)">전체 라이브러리를 훑어 합친다</text>
<text x="40" y="212" font-size="12" fill="var(--muted)">트리거: 7일 · 2시간 유휴(비활동)</text>

<g color="var(--core)">
  <path d="M356,130 L470,130" stroke="currentColor" stroke-width="2.2" marker-end="url(#sil2-a)"/>
</g>
<text x="413" y="120" font-size="11.5" text-anchor="middle" fill="var(--core)">보정</text>

<rect x="480" y="52" width="380" height="140" rx="11"
      fill="none" stroke="var(--line)" stroke-width="1.4"/>
<text x="500" y="80" font-size="13" font-weight="700" fill="var(--muted)">수명 주기</text>
<rect x="500" y="98" width="100" height="38" rx="8"
      fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.5"/>
<text x="550" y="122" font-size="12.5" text-anchor="middle" fill="var(--ok)">active</text>
<g color="var(--muted)">
  <path d="M606,117 L630,117" stroke="currentColor" stroke-width="2" marker-end="url(#sil2-a)"/>
</g>
<rect x="636" y="98" width="100" height="38" rx="8"
      fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.5"/>
<text x="686" y="122" font-size="12.5" text-anchor="middle" fill="var(--warn)">stale 30일</text>
<g color="var(--muted)">
  <path d="M742,117 L766,117" stroke="currentColor" stroke-width="2" marker-end="url(#sil2-a)"/>
</g>
<rect x="772" y="98" width="72" height="38" rx="8"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.5"/>
<text x="808" y="122" font-size="12" text-anchor="middle" fill="var(--edge)">archived</text>
<text x="500" y="162" font-size="12.5" fill="var(--danger)">자동 삭제는 없다</text>
</svg>
<figcaption>그림 2. 리뷰 포크는 <strong>늘리기만</strong> 하고, 큐레이터가 <strong>합쳐서</strong> 보정한다. 크론이 아니라 비활동으로 트리거된다.</figcaption>
</figure>


### 메모리만 엄격 검사를 거친다

스킬 콘텐츠 검사는 기본으로 꺼져 있다. 어차피 터미널 도구로 같은 코드를 돌릴 수 있으니 검사해 봐야 번거롭기만 하다는 이유다.

반면 메모리는 예외로 엄격 검사를 거친다. 이유는 명확하다 — 메모리는 시스템 프롬프트로 들어가기 때문이다.

> **핵심.** 보안 모델의 축이 콘텐츠 검열이 아니라 **어디로 흘러 들어가느냐**에 있다. 시스템 프롬프트에 닿는 경로만 조인다.
{: .prompt-info }

## 검증

2026-08-21 시점 v0.20.4 스냅숏을 클론해 확인했다.

| 구분 | 항목 |
| --- | --- |
| **확인함** | 주입이 없다는 주석 두 지점 · 카운터 두 개의 용도 · 포크 실행 경로 · 도구 배열 동일 전송 · 큐레이터 트리거 조건과 수명 주기 · 메모리 전용 엄격 검사 |
| **확인하지 못함** | 비용 26% 절감(저장소 주석의 수치를 옮긴 것) · 큐레이터의 실제 병합량 |

> **주의.** 26%는 내가 측정한 값이 아니라 저장소 주석에 적힌 값이다. 직접 재현하지 않았다.
{: .prompt-warning }

개선 효과를 판단할 신호도 찾아봤다. 가장 가까운 지표는 `reuse_after_patch` 하나뿐인데, 고친 뒤에 다시 쓰였는지를 세는 것이 전부다.

사용 기록은 `SKILL.md` 프론트매터가 아니라 별도 사이드카 파일에 쌓인다. 사용자가 쓴 내용과 운영 계측을 섞지 않으려는 선택이다. 카운터 갱신은 전부 best-effort라 사이드카가 깨져도 도구 호출은 멀쩡하다.

## 한계

가중치는 갱신되지 않는다. 검색 시점 학습이므로 전부 파일 입출력이다. 궤적을 JSONL로 내보내는 기능은 있지만 거기서 고리가 닫히지는 않는다. 학습과 재배포는 저장소 밖 사람의 몫이다.

설정 불일치도 하나 발견했다. 코드의 기본값은 메모리와 스킬 모두 10인데, 설정 예제 파일에는 15로 적혀 있다. 어느 쪽이 의도인지는 확인하지 못했다.

메모리와 스킬을 잇는 선은 임베딩이 아니라 어휘 겹침으로 만든다. 이름이 일치하면 가중치 6, 공통 토큰마다 1, 메모리당 최대 4개까지다. 비용이 0이고 결정론적이며 설명하기 쉽다는 장점이 있지만, 뜻은 같은데 단어가 다른 항목은 놓친다. 이 절충이 실사용에서 얼마나 손해인지는 측정하지 않았다.
