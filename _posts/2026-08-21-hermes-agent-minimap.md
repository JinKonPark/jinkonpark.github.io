---
title: "에이전트 하네스를 처음 열었을 때 어디부터 봐야 하는가"
date: 2026-08-21 13:20:00 +0900
categories: [AI Agents, Hermes Agent]
tags: [llm, agent, agent-harness, architecture]
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
처음 보는 에이전트 하네스 저장소를 열면 모듈 수백 개가 한꺼번에 눈에 들어온다. 어디부터 읽어야 할지 판단이 안 서서 파일 이름만 훑다가 닫는 일이 반복됐다. MIT 라이선스로 공개된 `NousResearch/hermes-agent`를 대상으로, 코드를 실제로 세어 가며 조감도를 그려 봤다.

## 문제

문서에 적힌 구조와 코드의 실제 구조가 어긋나 있으면 읽는 순서를 잡을 수 없다. 이 저장소에서도 그랬다. 저장소 문서에는 진입점 파일이 1만 2천 줄 규모라고 적혀 있지만, 실제로 세어 보니 7,419줄이었다. 가장 큰 파일은 문서가 언급조차 하지 않은 게이트웨이 쪽 25,304줄짜리였다.

> **이 문서가 정정하는 통념 하나.** 저장소 문서는 진입점이 1만 2천 줄이라고 적었지만 실측은 7,419줄이었다. 가장 큰 파일은 문서가 언급조차 하지 않은 게이트웨이 쪽 25,304줄짜리다.
{: .prompt-warning }

수치가 틀렸다는 것 자체는 큰 문제가 아니다. 그 수치를 근거로 "여기가 핵심이겠구나" 하고 읽기 시작하면 엉뚱한 곳에서 시간을 버린다는 게 문제다. 하루 약 188커밋이 들어오는 저장소에서 문서의 줄 번호는 금방 낡는다.

## 처음 시도한 접근

진입점부터 순서대로 읽어 내려갔다. 그런데 진입점으로 보이던 파일은 메서드 52개가 전부 forwarder였다. 실제 구현은 별도 패키지의 모듈 126개에 흩어져 있어서, 껍데기를 따라가는 동안 건진 게 거의 없었다.

이 저장소에서 "이름이 진입점처럼 생긴 파일"과 "실제로 로직이 있는 파일"은 다르다. 파일 크기나 이름으로 중요도를 짐작하는 방식은 여기서 통하지 않는다.

## 바꾼 설계

읽는 순서 대신 **구조의 허리**를 먼저 찾기로 했다. 저장소 문서에 이런 문장이 있다 — 가장자리는 넓게, 허리는 좁게(expansive at the edges, conservative at the waist).

세어 보니 그 문장이 정확했다.

| 층 | 개수 |
| --- | --- |
| 모델 프로바이더 | 32 |
| 메시징 플랫폼 | 21 |
| 스킬 | 181 |
| **코어 도구** | **54** |

모래시계 구조다. 바깥은 계속 늘어나지만 전부 코어 도구 54개라는 좁은 허리를 통과한다. 진입점도 마찬가지다. CLI, TUI, 메시징 게이트웨이, ACP 네 개가 전부 하나의 대화 루프 함수로 모인다.

<figure class="dg">
<svg viewBox="0 0 880 330" role="img" aria-label="가장자리는 넓고 허리는 좁은 모래시계 구조">
<defs>
  <marker id="mm1-a" markerWidth="9" markerHeight="9" refX="8" refY="3.2" orient="auto">
    <path d="M0,0 L8,3.2 L0,6.4 z" fill="currentColor"/>
  </marker>
</defs>
<text x="440" y="26" font-size="12.5" text-anchor="middle" fill="var(--muted)">가장자리는 넓다</text>
<rect x="40" y="40" width="180" height="52" rx="9"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.5"/>
<text x="130" y="63" font-size="12.5" text-anchor="middle" fill="var(--edge)">모델 프로바이더</text>
<text x="130" y="82" font-size="15" font-weight="700" text-anchor="middle" fill="var(--edge)">32</text>

<rect x="250" y="40" width="180" height="52" rx="9"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.5"/>
<text x="340" y="63" font-size="12.5" text-anchor="middle" fill="var(--edge)">메시징 플랫폼</text>
<text x="340" y="82" font-size="15" font-weight="700" text-anchor="middle" fill="var(--edge)">21</text>

<rect x="460" y="40" width="180" height="52" rx="9"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.5"/>
<text x="550" y="63" font-size="12.5" text-anchor="middle" fill="var(--edge)">스킬</text>
<text x="550" y="82" font-size="15" font-weight="700" text-anchor="middle" fill="var(--edge)">181</text>

<rect x="670" y="40" width="180" height="52" rx="9"
      fill="var(--fill-edge)" stroke="var(--edge)" stroke-width="1.5"/>
<text x="760" y="63" font-size="12.5" text-anchor="middle" fill="var(--edge)">진입점 CLI·TUI·GW·ACP</text>
<text x="760" y="82" font-size="15" font-weight="700" text-anchor="middle" fill="var(--edge)">4</text>

<g color="var(--muted)">
  <path d="M130,98 L400,150" stroke="currentColor" stroke-width="1.6"/>
  <path d="M340,98 L420,150" stroke="currentColor" stroke-width="1.6"/>
  <path d="M550,98 L462,150" stroke="currentColor" stroke-width="1.6"/>
  <path d="M760,98 L482,150" stroke="currentColor" stroke-width="1.6"/>
</g>

<rect x="330" y="156" width="220" height="66" rx="11"
      fill="var(--fill-core)" stroke="var(--core)" stroke-width="2.2"/>
<text x="440" y="182" font-size="13.5" font-weight="700" text-anchor="middle" fill="var(--core)">코어 도구 — 허리</text>
<text x="440" y="208" font-size="18" font-weight="700" text-anchor="middle" fill="var(--core)">54</text>

<g color="var(--core)">
  <path d="M440,228 L440,262" stroke="currentColor" stroke-width="2.2" marker-end="url(#mm1-a)"/>
</g>
<rect x="330" y="268" width="220" height="46" rx="9"
      fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.5"/>
<text x="440" y="296" font-size="12.5" text-anchor="middle" fill="var(--ok)">하나의 대화 루프 함수</text>

<text x="120" y="300" font-size="12.5" fill="var(--muted)">바깥은 계속 늘어나지만</text>
<text x="120" y="320" font-size="12.5" fill="var(--muted)">전부 이 허리를 통과한다</text>
</svg>
<figcaption>그림 1. 모래시계 구조. 프로바이더 32개를 다 읽을 이유가 없다 — <strong>허리를 먼저 읽고</strong> 필요한 가장자리만 따라가면 된다.</figcaption>
</figure>


이 구조를 파악하고 나니 읽는 순서가 저절로 정해졌다. 허리를 먼저 읽고, 필요한 가장자리만 따라가면 된다. 프로바이더 32개를 다 읽을 이유가 없다.

허리가 좁다는 증거는 디렉터리 배치에도 있다. 프로바이더 전용 디렉터리에는 파일이 3개뿐이고, 프로바이더 32개는 전부 플러그인 쪽에 있다. 코어는 개별 프로바이더에 의존하지 않는다.

### 도구는 등록만으로 보이지 않는다

허리를 읽다가 걸린 지점이 있다. 도구는 레지스트리에 등록했다고 해서 곧바로 모델 쪽에 보이지는 않는다. 관문 세 개를 모두 통과해야 한다.

```
registry.register()     # 1. 등록
  → toolsets 등재       # 2. 툴셋에 포함
  → check_fn 런타임 게이트  # 3. 실행 시점 조건 통과
```


<figure class="dg">
<svg viewBox="0 0 880 150" role="img" aria-label="도구가 모델에 보이기까지 통과해야 하는 관문 세 개">
<defs>
  <marker id="mm2-a" markerWidth="9" markerHeight="9" refX="8" refY="3.2" orient="auto">
    <path d="M0,0 L8,3.2 L0,6.4 z" fill="currentColor"/>
  </marker>
</defs>
<rect x="16" y="40" width="232" height="60" rx="10"
      fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.6"/>
<text x="132" y="66" font-size="12.5" text-anchor="middle" fill="var(--ok)">1. registry.register()</text>
<text x="132" y="87" font-size="11.5" text-anchor="middle" fill="var(--muted)">등록</text>
<g color="var(--core)"><path d="M254,70 L288,70" stroke="currentColor" stroke-width="2.2" marker-end="url(#mm2-a)"/></g>

<rect x="294" y="40" width="232" height="60" rx="10"
      fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.6"/>
<text x="410" y="66" font-size="12.5" text-anchor="middle" fill="var(--warn)">2. toolsets 등재</text>
<text x="410" y="87" font-size="11.5" text-anchor="middle" fill="var(--muted)">여기서 자주 빠진다</text>
<g color="var(--core)"><path d="M532,70 L566,70" stroke="currentColor" stroke-width="2.2" marker-end="url(#mm2-a)"/></g>

<rect x="572" y="40" width="232" height="60" rx="10"
      fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.6"/>
<text x="688" y="66" font-size="12.5" text-anchor="middle" fill="var(--warn)">3. check_fn 런타임 게이트</text>
<text x="688" y="87" font-size="11.5" text-anchor="middle" fill="var(--muted)">여기서도 자주 빠진다</text>
<g color="var(--core)"><path d="M810,70 L846,70" stroke="currentColor" stroke-width="2.2" marker-end="url(#mm2-a)"/></g>
<text x="862" y="66" font-size="12.5" text-anchor="middle" fill="var(--core)">모델</text>
<text x="862" y="84" font-size="12.5" text-anchor="middle" fill="var(--core)">스키마</text>
<text x="16" y="130" font-size="12.5" fill="var(--danger)">한 관문이라도 빠지면 도구는 모델에 실리지 않는다</text>
</svg>
<figcaption>그림 2. "등록했는데 왜 안 보이지" 싶을 때는 대개 <strong>2번이나 3번</strong>이 원인이다.</figcaption>
</figure>

한 관문이라도 빠지면 도구는 모델 스키마에 실리지 않는다. "등록했는데 왜 안 보이지" 싶을 때는 대개 2번이나 3번이 원인이다. 핸들러는 반드시 JSON 문자열을 반환해야 한다는 규약도 있어서, 어기면 반환값이 계약 위반 에러로 대체된다.

### 프롬프트 캐시를 건드리지 않는다는 원칙

이 저장소를 읽는 내내 반복해서 눈에 띈 설계 원칙이 하나 있다. 시스템 프롬프트의 앞부분을 바꾸지 않는다는 원칙이다.

여기서 파생된 선택들이 하나같이 일관적이다. 스킬 커맨드는 시스템 프롬프트가 아니라 사용자 메시지로 주입한다. 압축, 검색, 비전은 보조 클라이언트로 격리한다. 자가개선 루프조차 결과를 대화에 넣지 않고 파일에 써서 다음 세션에서 반영한다.

> **핵심.** 캐시 히트를 유지하겠다는 목적 하나가 구조 전반을 좌우한다. 기능을 하나 추가할 때도 "이게 앞쪽 컨텍스트를 건드리는가"가 판단 기준이 된다.
{: .prompt-info }

## 검증

실측은 저장소를 클론한 뒤 파일별로 줄 수를 세고 디렉터리별 파일 개수를 집계해서 했다. 2026-07-29 시점 v0.19.0 스냅숏 기준이다.

확인한 것과 확인하지 못한 것을 구분하면 이렇다.

- 확인함: 파일 줄 수, 디렉터리별 개수, forwarder 메서드 수, 진입점 4개가 같은 함수로 수렴한다는 사실
- 확인하지 못함: 캐시 히트율이 실제로 얼마나 유지되는지. 저장소 주석에 비용 절감 수치가 적혀 있지만 직접 측정하지 않았다

설정 로더가 셋이라는 점도 실측으로 확인했다. 같은 설정 파일을 세 함수가 각자 다른 기본값으로 읽는다. 어떤 설정 키가 특정 진입점에서만 먹힌다면 대개 여기가 원인이다.

## 한계

줄 번호는 스냅숏 기준이라 지금은 이미 어긋났을 가능성이 높다. 이 저장소를 다시 열 때는 줄 번호가 아니라 함수 이름과 심볼로 찾는 편이 안전하다.

모래시계 구조가 이 저장소만의 특성인지, 에이전트 하네스 전반이 수렴하는 형태인지는 이번 조사만으로 단정하기 어렵다. 다른 하네스도 같은 방식으로 그려서 비교해 봐야 답이 나온다.
