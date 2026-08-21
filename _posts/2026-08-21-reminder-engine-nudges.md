---
title: "에이전트 루프에 넛지를 흘려 넣는 규칙 엔진"
date: 2026-08-21 15:30:00 +0900
categories: [AI Agents, Engineering]
tags: [llm, agent, agent-harness, react-loop]
---

<style>
.dg { margin: 1.6rem 0; }
.dg svg { width: 100%; height: auto; display: block; }
.dg figcaption { font-size: 0.86rem; opacity: 0.75; margin-top: 0.6rem; line-height: 1.55; }
.dg {
  --core: #2f6f4f; --fill-core: #e8f3ec;
  --ok: #2f6f4f; --fill-ok: #e8f3ec;
  --warn: #9a6a12; --fill-warn: #fbf1dc;
  --stop: #a33a3a; --fill-stop: #f7e6e6;
  --muted: #6b7280; --fill-muted: #f1f2f4;
  --line: #9aa2ad; --ink: #2b2f36;
}
html[data-mode="dark"] .dg {
  --core: #7fc6a0; --fill-core: #1c2b23;
  --ok: #7fc6a0; --fill-ok: #1c2b23;
  --warn: #d9ad5c; --fill-warn: #2e2718;
  --stop: #e08c8c; --fill-stop: #2e1e1e;
  --muted: #9aa2ad; --fill-muted: #24262b;
  --line: #6b7280; --ink: #d7dae0;
}
</style>

에이전트는 도구를 부르고, 결과를 보고, 다시 도구를 부르는 **ReAct 루프**를 돈다. 사람이 옆에서 지켜본다면 "그 파일 벌써 세 번째 읽는데?" 하고 끼어들 만한 순간이 루프 안에는 꽤 자주 생긴다. 같은 에러를 계속 밟거나, 읽기만 반복하며 정작 손은 대지 않거나, 반복 횟수를 다 써 가는데 여전히 탐색만 하는 식이다.

사내 에이전트 하네스의 `ReminderEngine`은 그 "옆에서 한마디"를 자동화한 규칙 엔진이다. 이 글에서는 엔진이 언제 무엇을 근거로 넛지를 만들고, 그 넛지가 어떻게 모델에게 닿는지 정리한다.

> **세 줄 요약.** ReminderEngine은 매 도구 실행 직후 루프 상태를 훑어 "지금 모델에게 한마디 해야 하나"를 판정한다. 규칙은 5개(에러 복구, 탐색 반복, 도구 거부, 반복 횟수 경고, 할일 완료)이고, 조건은 전부 메시지 배열과 카운터만으로 따진다 — LLM을 다시 부르지 않는다. 넛지가 잔소리가 되지 않도록 쿨다운·총량 제한·one-shot이라는 세 겹 가드레일이 한곳에 모여 있다.
{: .prompt-info }

## 넛지가 왜 필요한가

**넛지(nudge)** 는 실행을 멈추지 않고 대화 기록에 짧은 안내 메시지를 하나 끼워 넣어 다음 턴의 판단만 살짝 밀어 주는 장치다. ReminderEngine은 그 한마디를 언제 던질지 정하는 판정기다.

<figure class="dg">
<svg viewBox="0 0 760 250" role="img" aria-label="ReAct 루프 안에서 ReminderEngine이 호출되는 위치를 보여주는 흐름도">
<defs>
<marker id="rn-ar" markerWidth="9" markerHeight="9" refX="7.5" refY="3" orient="auto">
<path d="M0 0 L7.5 3 L0 6 z" fill="var(--line)"/>
</marker>
</defs>
<rect x="24" y="30" width="130" height="46" rx="9" fill="var(--fill-muted)" stroke="var(--muted)" stroke-width="1.5"/>
<text x="89" y="59" font-size="13" text-anchor="middle" fill="var(--ink)">LLM 호출</text>

<rect x="208" y="30" width="130" height="46" rx="9" fill="var(--fill-muted)" stroke="var(--muted)" stroke-width="1.5"/>
<text x="273" y="59" font-size="13" text-anchor="middle" fill="var(--ink)">도구 실행</text>

<rect x="392" y="30" width="164" height="46" rx="9" fill="var(--fill-muted)" stroke="var(--muted)" stroke-width="1.5"/>
<text x="474" y="59" font-size="13" text-anchor="middle" fill="var(--ink)">messages에 결과 추가</text>

<rect x="392" y="122" width="164" height="62" rx="9" fill="var(--fill-core)" stroke="var(--core)" stroke-width="2.4"/>
<text x="474" y="147" font-size="13" text-anchor="middle" fill="var(--core)" font-weight="bold">ReminderEngine</text>
<text x="474" y="167" font-size="11.5" text-anchor="middle" fill="var(--core)">evaluate(loop_state)</text>

<rect x="188" y="122" width="150" height="62" rx="9" fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.5"/>
<text x="263" y="147" font-size="12.5" text-anchor="middle" fill="var(--ok)">넛지 있으면</text>
<text x="263" y="167" font-size="11.5" text-anchor="middle" fill="var(--ok)">append_nudge()</text>

<rect x="600" y="122" width="136" height="62" rx="9" fill="none" stroke="var(--muted)" stroke-width="1.4" stroke-dasharray="5 4"/>
<text x="668" y="147" font-size="12" text-anchor="middle" fill="var(--muted)">LoopState</text>
<text x="668" y="166" font-size="11" text-anchor="middle" fill="var(--muted)">messages · iteration</text>

<path d="M154 53 H204" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar)"/>
<path d="M338 53 H388" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar)"/>
<path d="M474 76 V118" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar)"/>
<path d="M392 153 H342" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar)"/>
<path d="M188 153 H89 V80" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar)"/>
<path d="M600 153 H560" stroke="var(--line)" stroke-width="1.4" stroke-dasharray="5 4" fill="none" marker-end="url(#rn-ar)"/>

<text x="490" y="104" font-size="11" fill="var(--muted)">매 반복</text>
<text x="263" y="212" font-size="11" text-anchor="middle" fill="var(--muted)">다음 반복으로 — 넛지는 대화 기록에 남은 채로</text>
</svg>
<figcaption>그림 1. 엔진이 불리는 지점은 도구 실행 결과가 <code>messages</code>에 붙은 직후이자 다음 LLM 호출 직전인 한 곳뿐이다.</figcaption>
</figure>

호출부는 루프 끝에서 이런 모양이다.

```python
loop_state.messages = messages
loop_state.iteration = iteration
loop_state.max_iterations = self.max_iterations
loop_state.force_answer = force_answer

if self.config.nudges_enabled:
    for nudge in reminder_engine.evaluate(loop_state):
        self._append_nudge(messages, nudge)
```

여기서 두 가지를 눈여겨볼 만하다. 첫째, 엔진은 **메시지를 직접 건드리지 않는다**. 문자열 목록만 돌려주고 실제 주입은 호출자가 한다. 판정과 주입을 갈라 둔 덕분에 엔진은 순수 함수에 가깝고 테스트하기도 쉽다.

둘째, `nudges_enabled`라는 스위치가 앞을 막고 있다.

> **기본값이 꺼져 있다.** 이 스위치의 기본값은 `False`다. 넛지가 하나 붙을 때마다 모델이 그 안내를 처리하느라 턴을 더 쓰기 때문이다. 그래서 넛지가 필요한 실행에서만 따로 켠다.
{: .prompt-warning }

## 판정 재료 — LoopState

`LoopState`는 ReAct 반복 사이에 유지되는 가변 상태를 담는 데이터 클래스다. 엔진이 참조하는 필드는 이 정도다.

| 필드 | 쓰임 |
| --- | --- |
| `messages` | 누적된 대화 기록. 마지막 도구 에러와 연속 읽기 횟수를 여기서 역순으로 훑어 센다 |
| `iteration` / `max_iterations` | 현재 반복 번호와 상한. 쿨다운 계산과 반복 경고의 기준이다. 상한 기본값은 35 |
| `force_answer` | 반복을 다 써서 도구 없이 답만 쓰는 마무리 단계라는 표시 |
| `tool_denied` | 승인 규칙이 도구 호출을 거부하면 켜진다. 넛지를 보낸 뒤 엔진이 직접 끈다 |
| `todo_manager` | 할일 목록 핸들. 완료 여부를 여기에 물어본다 |
| `all_todos_complete_nudged` | 할일 완료 넛지가 실행당 한 번만 나가도록 막는 빗장 |
| `benchmark_mode` | 벤치마크 실행 표시. 탐색을 재촉하는 넛지를 통째로 끈다 |

모두 루프가 이미 들고 있는 값이다. 판정하려고 LLM을 한 번 더 부르거나 외부 상태를 조회하지 않는다. 그래서 `evaluate` 한 번은 사실상 공짜다.

## evaluate — 5개 규칙을 순서대로

진입점부터 보자.

```python
def evaluate(self, ctx: LoopState) -> list[str]:
    if ctx.force_answer:
        return []

    nudges: list[str] = []
    self._check_error_recovery(ctx, nudges)      # 1
    self._check_exploration_spiral(ctx, nudges)  # 2
    self._check_tool_denied(ctx, nudges)         # 3
    self._check_iteration_warning(ctx, nudges)   # 4
    self._check_todos(ctx, nudges)               # 5
    return nudges
```

맨 앞의 `force_answer` 가드가 중요하다. 이미 마무리 답변을 쓰는 중이라면 "더 조사해라", "다르게 시도해라" 같은 안내는 방해만 된다. 그래서 이 단계에서는 아무 넛지도 내보내지 않는다.

다섯 규칙은 **서로 배타적이지 않다**. 조건이 겹치면 한 반복에 넛지가 둘 이상 나가고, 목록 순서가 곧 주입 순서다. 그런데도 실제로 겹치는 일이 드문 건 뒤에서 볼 가드레일 덕분이다.

<figure class="dg">
<svg viewBox="0 0 760 400" role="img" aria-label="evaluate가 force_answer 가드를 지나 다섯 규칙을 차례로 검사하는 흐름도">
<defs>
<marker id="rn-ar2" markerWidth="9" markerHeight="9" refX="7.5" refY="3" orient="auto">
<path d="M0 0 L7.5 3 L0 6 z" fill="var(--line)"/>
</marker>
</defs>
<rect x="262" y="12" width="186" height="40" rx="9" fill="var(--fill-core)" stroke="var(--core)" stroke-width="2.2"/>
<text x="355" y="37" font-size="13" text-anchor="middle" fill="var(--core)" font-weight="bold">evaluate(ctx)</text>

<path d="M355 52 V80" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>
<path d="M283 101 L355 82 L427 101 L355 120 Z" fill="var(--fill-warn)" stroke="var(--warn)" stroke-width="1.5"/>
<text x="355" y="105" font-size="11.5" text-anchor="middle" fill="var(--warn)">force_answer?</text>

<path d="M427 101 H512" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>
<text x="468" y="94" font-size="11" text-anchor="middle" fill="var(--muted)">예</text>
<rect x="516" y="81" width="126" height="40" rx="9" fill="none" stroke="var(--muted)" stroke-width="1.4" stroke-dasharray="5 4"/>
<text x="579" y="106" font-size="11.5" text-anchor="middle" fill="var(--muted)">빈 목록 반환</text>

<path d="M355 120 V140" stroke="var(--line)" stroke-width="1.5" fill="none"/>
<text x="368" y="136" font-size="11" fill="var(--muted)">아니오</text>
<path d="M355 140 H190 V148" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>

<rect x="64" y="150" width="252" height="34" rx="7" fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.4"/>
<text x="78" y="172" font-size="12" fill="var(--ok)">① _check_error_recovery</text>
<text x="330" y="172" font-size="11" fill="var(--muted)">직전 도구 결과가 Error로 시작하나</text>

<rect x="64" y="197" width="252" height="34" rx="7" fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.4"/>
<text x="78" y="219" font-size="12" fill="var(--ok)">② _check_exploration_spiral</text>
<text x="330" y="219" font-size="11" fill="var(--muted)">읽기 전용 호출만 5턴 연속인가</text>

<rect x="64" y="244" width="252" height="34" rx="7" fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.4"/>
<text x="78" y="266" font-size="12" fill="var(--ok)">③ _check_tool_denied</text>
<text x="330" y="266" font-size="11" fill="var(--muted)">tool_denied 플래그가 켜졌나</text>

<rect x="64" y="291" width="252" height="34" rx="7" fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.4"/>
<text x="78" y="313" font-size="12" fill="var(--ok)">④ _check_iteration_warning</text>
<text x="330" y="313" font-size="11" fill="var(--muted)">남은 반복이 임계값 이하인가</text>

<rect x="64" y="338" width="252" height="34" rx="7" fill="var(--fill-ok)" stroke="var(--ok)" stroke-width="1.4"/>
<text x="78" y="360" font-size="12" fill="var(--ok)">⑤ _check_todos</text>
<text x="330" y="360" font-size="11" fill="var(--muted)">할일이 전부 완료됐나</text>

<path d="M190 184 V197" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>
<path d="M190 231 V244" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>
<path d="M190 278 V291" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>
<path d="M190 325 V338" stroke="var(--line)" stroke-width="1.5" fill="none" marker-end="url(#rn-ar2)"/>
</svg>
<figcaption>그림 2. 다섯 규칙은 배타적이지 않아 한 반복에 여러 개가 함께 나가기도 한다. 겹침을 실제로 줄이는 건 순서가 아니라 <code>_try_fire</code>의 가드레일이다.</figcaption>
</figure>

### ① 에러 복구 — 에러 종류를 가려내 맞는 조언을 준다

가장 손이 많이 간 규칙이다. 먼저 `messages`를 뒤에서부터 훑어 **가장 최근 도구 메시지**를 찾고, 그 내용이 `"Error"`로 시작하는지 본다. 도구 메시지를 하나 찾으면 거기서 멈춘다 — 더 이전은 보지 않으니 "직전 호출이 실패했나"만 따지는 셈이다.

에러가 있으면 연속 에러 카운터를 1 올린 뒤 두 갈래로 나뉜다.

**연속 2회 이상이면 반복 에러 경고**를 보낸다(임계값 기본 2). 문구는 "같은 방식으로 재시도하지 말고 전략을 완전히 바꾸거나, 이미 가진 정보로 답하라"는 쪽이다.

**첫 에러라면 종류를 가려내** 거기 맞는 조언을 보낸다. 분류는 정규식 7개를 위에서부터 차례로 맞춰 보는 단순한 방식이다.

| 카테고리 | 매칭 신호(일부) | 조언 요지 |
| --- | --- | --- |
| `permission_error` | 403, forbidden, access denied | 같은 명령 재시도 금지. 쓰기 권한 있는 경로를 찾아라 |
| `edit_mismatch` | old_content, content mismatch | 파일이 바뀐 상황이다. 다시 읽고 정확한 텍스트로 재시도 |
| `file_not_found` | 404, no such file | 경로를 추측하지 말고 목록 조회로 먼저 찾아라 |
| `timeout` | timed out, deadline exceeded | 범위를 좁혀라. 전체 대신 특정 디렉터리로 |
| `rate_limit` | 429, quota | 잠시 기다리고 동시 실행을 줄여라 |
| `syntax` | syntax error, JSONDecodeError | 파일 현재 상태를 읽고 문법을 고쳐 재시도 |
| `generic_read_file` | read_file, encoding | 인코딩 문제면 다른 방법, 아니면 다른 출처 |
| `unknown` | (어디에도 안 걸림) | 범용 폴백 — "에러를 읽고 원인을 고친 뒤 재시도" |

순서가 곧 우선순위라서 위에 있는 패턴이 먼저 걸린다. 예를 들어 `"403 forbidden ... timeout"` 같은 문자열은 `permission_error`로 분류된다.

이 분류 넛지에는 별도 예산이 하나 더 걸려 있다. 에러 넛지 발화 수가 상한(기본 3)에 닿으면 그 뒤로는 분류 넛지를 아예 만들지 않는다. 도구 에러가 잦은 작업에서 대화 기록이 넛지로 도배되지 않게 하려는 장치다.

도구 에러가 없으면 연속 카운터는 0으로 초기화된다. "연속"이 말 그대로 연속을 뜻하게 해 주는 부분이다.

### ② 탐색 반복 — 읽기만 하고 손을 안 댈 때

읽기 전용 도구만 계속 부르는 상태를 잡아낸다. 읽기 전용으로 치는 도구는 다섯 개다.

```python
READ_OPS = frozenset({
    "read_file", "fetch_url", "web_extract", "web_search", "browse_webpage",
})
```

세는 방법은 이렇다. 메시지를 뒤에서부터 보면서 `tool`·`user`·`system` 역할은 건너뛰고, assistant 턴을 만나면 그 턴의 도구 호출이 **전부** `READ_OPS`에 속하는지 확인한다. 쓰기 계열이 하나라도 섞이면 거기서 세기를 멈춘다. 그렇게 센 연속 읽기 턴이 임계값(기본 5) 이상이면 넛지가 나간다.

넛지 문구는 몇 번 읽었는지 숫자로 알려 주고 선택지를 셋 제시한다 — 정보가 충분하면 진행하고, 불명확하면 사용자에게 묻고, 막혔으면 무엇에 막혔는지 설명하라.

> **벤치마크에서는 끈다.** `benchmark_mode`가 켜져 있으면 이 규칙은 곧바로 빠져나온다. 벤치마크에는 원래 자료를 한참 찾아야 풀리는 문제가 많아서, 진행을 재촉하는 넛지가 오히려 탐색을 일찍 끊어 정답률을 떨어뜨린다.
{: .prompt-warning }

### ③ 도구 거부 — 승인 규칙에 막혔을 때

제일 짧은 규칙이다. `tool_denied`가 켜져 있으면 "같은 호출을 다시 시도하지 말고, 왜 거부됐는지 생각해 접근을 바꿔라. 모르겠으면 사용자에게 물어봐라"는 넛지를 보낸다.

특이한 점은 넛지를 보낸 뒤 **엔진이 `tool_denied`를 직접 `False`로 되돌린다**는 것이다. 이 플래그는 도구 실행 쪽에서 켜 주는 값이라, 엔진이 끄지 않으면 계속 켜진 채로 남아 매 반복 같은 넛지를 또 내보내려 한다. 켠 쪽이 아니라 읽어 쓴 쪽이 치우는 구조다.

### ④ 반복 횟수 경고 — 마감이 다가올 때

남은 반복이 얼마 없다고 미리 알려 답을 정리하게 만드는 규칙이다. 임계값 계산은 이렇다.

```python
remaining = ctx.max_iterations - ctx.iteration
warning_threshold = max(3, int(ctx.max_iterations * self._iteration_warning_ratio))
near_limit = remaining <= warning_threshold
not_final = remaining > 1
```

경고 비율 기본값은 0.3이고 `max_iterations` 기본값은 35다. 그러면 임계값이 `max(3, 10) = 10`이 되어 25번째 반복부터 경고가 시작된다. `max(3, ...)`이 붙은 이유는 반복 상한이 아주 작은 실행(예: 5회)에서도 최소 세 번의 여유는 남기고 알려 주려는 것이다.

`not_final` 조건도 눈여겨볼 만하다. 남은 반복이 1이면 경고를 보내지 않는다. 이미 마지막 턴인데 "답을 정리하라"고 말해 봐야 그 말을 처리할 턴 자체가 없다.

### ⑤ 할일 완료 — 마무리 신호

할일 목록이 있고, 그 목록이 비어 있지 않으며, 미완료 항목이 하나도 없을 때 딱 한 번 나가는 넛지다. 내용은 "이제 도구 호출 없이, 사용자가 요청한 실제 결과물을 평문으로 써라"는 마무리 지시다. 작업 회고가 아니라 요청받은 내용 자체를 쓰라고 못 박은 점이 핵심이다.

중복 방지가 이중으로 걸려 있다. 엔진 안에서는 `one_shot=True`로, 바깥에서는 `all_todos_complete_nudged` 플래그로 막는다. 앞쪽은 엔진 인스턴스가 살아 있는 동안, 뒤쪽은 `LoopState`가 살아 있는 동안 유효하다.

## 가드레일 — 넛지가 잔소리가 되지 않게

다섯 규칙은 넛지를 만들 때 모두 `_try_fire`를 거치고, 억제 로직은 거기 한곳에 모여 있다.

```python
def _try_fire(self, name, iteration, content, *, max_count=None, one_shot=False):
    if not content:
        return None

    cooldown = self._cooldown_iterations
    last = self._last_fired.get(name, -cooldown - 1)
    if iteration - last < cooldown:                   # ① 쿨다운
        return None

    count = self._fire_counts.get(name, 0)
    if one_shot and count > 0:                        # ② 1회 제한
        return None
    if max_count is not None and count >= max_count:  # ③ 총량 제한
        return None

    self._last_fired[name] = iteration
    self._fire_counts[name] = count + 1
    return content
```

| 장치 | 기준 | 기본값 | 막는 상황 |
| --- | --- | --- | --- |
| 쿨다운 | 같은 이름 넛지의 최소 발화 간격(반복 수) | 3 | 같은 조언이 매 턴 되풀이되는 상황 |
| 총량 제한 | 실행 전체에서 그 이름의 총 발화 횟수 | 규칙별 | 긴 실행에서 같은 종류가 쌓이는 상황 |
| one-shot | 총량 제한을 1로 준 것과 같다 | — | 일회성 신호가 다시 나가는 상황 |

키가 **넛지 이름**이라는 점이 중요하다. 에러 분류 넛지는 `f"error_{category}"`처럼 카테고리마다 이름이 달라, 권한 에러와 타임아웃 에러는 서로의 쿨다운을 잡아먹지 않는다. 반면 반복 횟수 경고는 이름이 하나뿐이라 3반복에 한 번꼴로만 나간다.

초기값 `-cooldown - 1`도 작지만 요긴한 장치다. 이렇게 두면 `iteration=0`에서도 `0 - (-4) = 4 ≥ 3`이 되어 첫 발화는 항상 통과한다.

생성자 파라미터는 모두 설정 객체에서 받아 온다.

```python
reminder_engine = ReminderEngine(
    cooldown_iterations=cfg.reminder_cooldown_iterations,          # 3
    stagnation_threshold=cfg.reminder_stagnation_threshold,        # 5
    error_repeat_threshold=cfg.reminder_error_repeat_threshold,    # 2
    iteration_warning_ratio=cfg.reminder_iteration_warning_ratio,  # 0.3
    max_error_nudges=cfg.reminder_max_error_nudges,                # 3
)
```

엔진은 실행 하나당 한 번 만들어진다. 카운터가 인스턴스 필드에 들어 있으니 **가드레일의 유효 범위도 곧 한 번의 실행**이다.

## 문구는 어디서 오나 — reminders.md

넛지 문구는 코드에 문자열로 박혀 있지 않고 마크다운 파일 한 곳에 모여 있다.

```markdown
--- iteration_warning ---
You are at iteration {current}/{max}. {remaining} iterations remaining.
Start forming your answer. If you have enough information, use it to answer NOW.
```

`--- 이름 ---` 구분선으로 섹션을 나누고, `get_reminder("이름", key=value)`로 꺼내면서 `{key}` 자리를 채운다. 파싱 결과는 `functools.lru_cache`에 담아 두어 파일은 한 번만 읽는다. 이름이 없으면 경고 로그를 남기고 빈 문자열을 돌려주는데, `_try_fire`가 맨 앞에서 빈 문자열을 걸러 내니 넛지가 조용히 사라질 뿐 예외로 터지지는 않는다.

> **문구를 고치면 모델 동작이 바뀐다.** `reminders.md`는 코드가 아니라 프롬프트다. 여기 한 줄을 손대면 리팩터링이 아니라 모델에게 주는 지시를 바꾸는 셈이다. 요청 범위 밖이면 건드리지 말고, 바꿨다면 보고와 커밋 메시지에서 따로 짚는다.
{: .prompt-danger }

## 넛지가 모델에게 닿는 방식

엔진이 돌려준 문자열은 `append_nudge`를 거쳐 `<system-reminder>` 태그로 감싼 `user` 메시지가 된다. 내부 분기용으로 메시지 클래스 표식도 함께 붙는다.

```python
msg: ChatMessage = {
    "role": "user",
    "content": wrap_system_reminder(content),
    "_msg_class": MessageClass.NUDGE.value,
}
```

태그로 감싸는 이유는 경계를 분명히 하려는 것이다. 접두사만 붙이면 모델이 도구 결과나 사용자 발화와 섞어 읽을 여지가 있지만, 여는 태그와 닫는 태그가 있으면 어디까지가 시스템의 말인지 헷갈릴 일이 없다.

여기에 더해, 도구 결과 바로 뒤에 붙은 넛지는 **별도의 사용자 턴이 아니라 도구 턴의 꼬리로** 그려진다. 디스패치가 꼬리 표식을 달아 주면 provider 어댑터가 각자 형식에 맞게 렌더링한다. Anthropic에서는 `tool_result` 블록 뒤에 같은 사용자 메시지 안의 text 블록으로 붙고, OpenAI 계열은 표식만 떼어 내고 기존 구조를 그대로 둔다.

수명 규칙도 코드에 못 박혀 있다. 한 번 주입한 넛지는 실행이 끝날 때까지 대화 기록에 남는다. 뒤늦게 빼면 provider의 프롬프트 캐시가 그 지점부터 무효가 되고, 모델이 이미 읽은 맥락과도 어긋나기 때문이다. 대신 넛지 표식이 붙은 메시지는 세션 스냅숏에 저장되지 않아 다음 실행까지 따라가지는 않는다. 같은 넛지가 또 나가는 건 삭제가 아니라 엔진의 쿨다운이 막는다.

## 엔진 밖의 넛지

헷갈리기 쉬운 부분이라 짚어 둔다. `append_nudge`를 부르는 곳이 ReminderEngine뿐은 아니다.

| 넛지 | 주체 | 발동 조건 |
| --- | --- | --- |
| 둠 루프 리다이렉트 | 둠 루프 감지기 | 같은 도구 호출 패턴이 반복될 때. 3단계(Redirect / Notify / ForceStop)로 올라간다 |
| 미완료 할일 재촉 | 실행기의 할일 처리부 | 벤치마크 정책에서 도구 호출 없이 턴을 끝내려 할 때 |
| 최종 답변 검증 | 실행기의 답변 확정부 | 답변 후보가 형식·완결성 검증을 통과하지 못할 때 |

ReminderEngine은 이 중 **정기 점검**을 맡는 쪽이다. 매 반복 같은 자리에서 상태를 훑는다. 나머지는 특정 사건이 터진 자리에서 그때그때 넛지를 만든다. 넛지가 왜 나왔는지 추적할 때는 이 구분부터 확인하는 게 빠르다.

## 마무리

> **한눈에 보기.** ReAct 루프에서 도구 결과가 붙은 직후 `evaluate`가 한 번 불린다. 규칙은 5개 — 에러 복구(종류별 조언 + 연속 경고), 탐색 반복(읽기만 5턴), 도구 거부, 반복 횟수 경고(남은 30%), 할일 완료. 가드레일은 3겹 — 쿨다운 3반복, 이름별 총량 제한, one-shot. 모두 `_try_fire` 한곳에 있고 유효 범위는 실행 하나다. 문구는 코드가 아니라 데이터이고, 스위치 기본값이 꺼짐이라 켜지 않으면 판정조차 하지 않는다.
{: .prompt-tip }
