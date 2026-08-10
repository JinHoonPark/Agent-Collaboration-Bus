# M0 실측 결과

스펙 `ACB-spec-v0.2.md` 14장의 미결 항목에 대한 사실 확인 기록.
관찰한 사실과 그로부터의 추정을 구분해 표기한다.

> **M0 종료 — 2026-08-11.** 실측 6건 전부 완료. 설계상 미결 없음.
> 결과는 스펙 v0.2 에 반영했다(0.1 절의 변경 요약 참조). 다음 단계는 M1 브로커 구현이다.
>
> **범위 주의:** 스펙 13장의 원래 M0 범위에는 "공유 폴더 메일박스로 워크스페이스 2개
> 왕복"이라는 미니 구현이 포함돼 있었으나 **수행하지 않았다.** 브로커가 생기면 폐기될
> 200줄이라 M1 으로 흡수하기로 했다. 이 문서는 **실측 결과만** 다룬다.
> 그 결과 **에이전트 두 개가 요청·응답을 완주한 적이 아직 없으며**, 여기서 측정한 것은
> 전부 "외부 스크립트 -> 훅 -> 에이전트 세션 1개" 구조다. 넘어간 위험은 스펙 14장
> 잔여 항목표의 앞 세 줄에 있다.
>
> **프로젝트 전제 확인:** 훅으로 주입한 텍스트가 확률적 시스템인 모델의 행동을 실제로
> 바꾼다는 것이 두 클라이언트 모두에서 증명됐다. 워크스페이스 밖에 둔 난수 토큰을 모델이
> 정확히 산출하는 방식으로 기계적으로 확인했다.
>
> **전제 유지:** 완전 idle 인 에이전트 세션을 외부에서 깨워 작업을 시작시키는 것은
> Claude Code 에서 여전히 불가능하다. MCP elicitation 은 사람을 깨우지 에이전트를
> 깨우지 않는다. 스펙 1장의 "완전 자동화는 비목표"는 그대로다.
>
> M1 에서 확인해야 할 잔여 항목은 스펙 14장 말미 표에 정리했다.

| # | 대상 | 항목 | 상태 |
|---|---|---|---|
| 1 | Claude Code | Stop 훅 turn 연장이 모델 행동을 바꾸는가 | **PASS** (2026-08-10) |
| 2 | Claude Code | PostToolUse `additionalContext` 실효성 | **PASS** (2026-08-10) |
| 3 | Claude Code + Codex | MCP sampling / elicitation 지원 여부 | **완료** — 양쪽 다 sampling 미지원 / elicitation 지원 (2026-08-10) |
| 4 | Codex | App Server `turn/start` 의 TUI 렌더링 | **PASS — 결과 (a)** (2026-08-10) |
| 5 | Codex | WebSocket 없이 로컬 App Server 접속 | 부분 확인 (2026-08-10) |
| 6 | Codex | **(신규)** Codex 훅이 모델 컨텍스트에 주입 가능한가 | **PASS** (2026-08-10) |

6번은 원래 스펙에 없던 항목이다. M0-2 를 마친 뒤, "Claude 는 훅만으로 되는데 Codex 도
훅만으로 맞추면 되지 않나"라는 검토에서 도출됐다. 자세한 배경은 M0-4 절 말미에 있다.

---

## M0-1 — Claude Code Stop 훅 턴 연장 (PASS)

### 검증 일시 / 환경

- 2026-08-10, Windows 11 Home 10.0.26200
- Node.js v25.8.1
- 테스트 워크스페이스: `D:\Projects\Utils\Agent-Collaboration-Bus\M0\stop-hook-test\`
- 테스트 세션 ID: `0f95b824-f1d2-4926-86aa-6f618f261f16` (R1~R3 동일 세션)

### 검증 방법

Stop 훅이 `{"decision":"block","reason":"..."}` 을 stdout 으로 반환해 턴을 연장하고,
reason 에 "inbox.json 을 읽고 `body.echo` 값을 outbox.json 에 옮겨 적으라"는 지시를 넣었다.

`body.echo` 는 seed 시점에 생성한 12자리 난수(6바이트 hex 대문자)다.
**모델이 파일을 읽지 않고는 알 수 없는 값이므로, echo 일치는 "실제로 읽고 수행했다"는
기계적 증거가 된다.** 훅 호출은 전부 `hook.log` 에 JSON Lines 로 기록했다.

사전에 가짜 stdin JSON 으로 훅의 4개 분기(inbox 없음 / 미통지 있음 / 미통지 없음 /
`stop_hook_active=true`)를 오프라인 검증해, 스크립트 버그로 인한 실패 가능성을 배제한 뒤
실세션 테스트를 진행했다.

### 관찰된 사실

**R1 — 기본 (턴 종료 직후, 컨텍스트 비어 있음)**

| 시각 (UTC) | 사건 |
|---|---|
| (사전) | `msg_test_R1_d5a145` 적재, echo `4D10C313EF72` |
| 14:16:56.844 | `stop_hook_active=false` -> `decision=block` |
| 14:17:07.647 | `stop_hook_active=true` -> `allow`. 턴 정상 종료 |

모델은 되묻지 않고 한 턴 안에 `inbox.json` 을 읽고 `outbox.json` 을 작성했다.
echo 일치. 판정 PASS.

**R2 — 현실 타이밍 (긴 작업 도중 메시지 도착)**

테스트 세션에 `.acb-test` 의 .js 파일 4개를 읽고 함수 단위 표를 만드는 작업을 지시하고,
작업 도중 메시지를 적재했다.

| 시각 (UTC) | 사건 |
|---|---|
| 14:21:40 | `msg_test_R2_013a0d` 적재, echo `7D5A21F0774B` (모델 작업 중) |
| 14:23:17.313 | 표 작성 완료 후 턴 종료 시도 -> `decision=block` |
| 14:23:34.188 | `stop_hook_active=true` -> `allow`. 턴 정상 종료 |

echo 일치. 판정 PASS. 추가로 관찰된 것:

- 모델은 **원래 지시받은 표 작성을 끝까지 완주한 뒤** 훅 지시를 처리했다.
  중간에 원래 작업을 버리고 갈아타지 않았다.
- 응답 말미에 "이전 R1 라운드의 outbox.json 은 reset.js 로 이미 삭제된 상태였습니다"라고
  덧붙였다. 이는 그 턴에 읽은 `reset.js` 코드에서 추론한 내용으로, 긴 작업으로 컨텍스트가
  찬 상태에서도 기계적 처리가 아니라 상황을 이해한 처리를 했음을 보여준다.

**R3 — 메시지 없을 때 정상 종료**

`inbox.json` 은 존재하되 미통지 메시지가 0건인 상태(ACB 실사용의 평상시 모습)에서
사소한 작업을 지시했다.

| 시각 (UTC) | 사건 |
|---|---|
| 14:26:19.835 | `stop_hook_active=false` -> `allow` (미통지 메시지 없음) |

훅 개입 없이 턴이 즉시 종료됐고 `outbox.json` 도 변경되지 않았다. 중복 배달 없음. PASS.

### 확정된 사실 4가지

1. **Stop 훅의 `decision:"block"` 은 턴을 연장하며, 모델은 주입된 reason 을 실제로 수행한다.**
   R1, R2 모두 난수 토큰이 정확히 일치했다.
2. **훅 주입은 모델이 진행 중이던 원래 작업을 방해하지 않는다.** 원래 작업을 완주한 뒤
   주입 지시를 처리했다.
3. **`stop_hook_active` 는 실제로 `true` 로 전달된다.** R1, R2 모두 두 번째 호출에서 확인됐다.
   스펙이 상정한 무한 루프 방지 전략이 추측이 아니라 확인된 사실이 됐다.
4. **메시지가 없으면 훅은 조용히 통과한다.** 평상시 세션 동작을 방해하지 않는다.

### 부수적으로 확인된 운용상 사실

- **주입 텍스트는 사용자 화면에 그대로 표시된다.** 단 Claude Code 는 이를
  `Stop hook error:` 라는 접두사와 함께 표시한다. 정상 동작이 에러처럼 보이므로
  실사용 시 사용자가 오작동으로 오해할 소지가 있다.
  -> 주입 문구를 `[ACB]` 로 시작하게 해 구분되도록 한다 (스펙 7.4 템플릿이 이미 그러함).
- **배달 지연은 R2 기준 97초였다** (적재 14:21:40 -> 배달 14:23:17).
  Stop 훅은 현재 턴이 끝나야 배달하므로, 지연은 진행 중인 작업의 길이에 비례한다.
- 긴 주입 텍스트와 모델의 표 출력이 터미널에서 일부 잘려 보이는 현상이 있었다.
  파일 내용은 정상이므로 렌더링 문제로 판단되며 ACB 기능과 무관하다.
- 파일 읽기/쓰기 권한 프롬프트는 뜨지 않았다. 테스트 세션의 권한 모드에 따른 것으로
  보이며, ACB 실사용에서는 MCP 툴 호출 경로라 이 관찰은 그대로 적용되지 않는다.

### 검증되지 않은 것 (한계)

- **MCP 툴 호출로의 일반화는 확인되지 않았다.** 이번 주입 지시는 파일 읽기/쓰기였고,
  실제 ACB 는 `acb_inbox` MCP 툴 호출을 지시한다. 툴 호출이 파일 조작보다 어렵다고 볼
  근거는 없으나 이는 추정이며, M1 통합 테스트에서 확인해야 한다.
- **연장된 턴 도중 도착한 메시지의 처리**는 테스트하지 않았다. 훅 로직상
  `stop_hook_active=true` 인 동안에는 추가 주입을 하지 않으므로 다음 턴으로 밀린다.
- 단일 세션, 단일 메시지, 3회 시행이다. 확률적 시스템이므로 100% 재현을 보장하지 않는다.
- 여러 메시지 동시 배달(N건)은 테스트하지 않았다.

### 스펙에 미치는 영향

**설계 변경 없음.** 스펙 7.2 의 `wake_mode` 열거에서 `hook` 모드가 유효함이 확인됐고,
4장 스키마와 6장 툴 시그니처는 그대로 간다.

M0-1 이 PASS 이므로 **ACB 프로젝트의 전제가 성립한다.** spawn 전용 설계로의 축소는
불필요하다.

M0-2(PostToolUse `additionalContext`)의 가치가 이번 측정으로 구체화됐다.
성공하면 배달 지연이 97초 수준에서 툴 호출 1회 간격(수 초)으로 줄고,
실패하면 ACB 의 배달 지연은 "턴 경계"로 고정된다.

### 재현 방법

`D:\Projects\Utils\Agent-Collaboration-Bus\M0\stop-hook-test\README.md` 참조.
훅 로그 원본은 같은 디렉터리의 `.acb-test\hook.log` 및 `.acb-test\hook.log.*.bak` 에 있다.

---

## M0-2 — Claude Code PostToolUse `additionalContext` (PASS)

### 검증 일시 / 환경

- 2026-08-10, 환경은 M0-1 과 동일
- 테스트 세션 ID: `e9bf173f-1a84-4c22-9260-a979b88f7549` (훅 추가 후 재시작한 세션)
- PostToolUse 훅 등록 시 `matcher: "*"` 사용

### 검증 방법 — 토큰을 워크스페이스 밖에 둔다

M0-1 과 결정적으로 다른 설계를 썼다. 정답 토큰을 테스트 워크스페이스 **밖**
(스크래치패드의 `m2-state.json`)에 보관하고, 워크스페이스 안에는 토큰을 두지 않았다.
검증 직전 워크스페이스 전수 검색으로 토큰 문자열 `M2-3787AF252C5A` 의 부재를 확인했다.

이렇게 하면 모델이 워크스페이스 파일을 전부 읽어도 토큰을 알 수 없다.
**따라서 모델이 토큰을 정확히 산출하면, 그 정보의 유일한 유입 경로는
`additionalContext` 이며 이는 컨텍스트 진입의 반박 불가능한 증거가 된다.**

측정은 두 단계로 설계했다.

| 단계 | 확인 내용 | ACB 에서의 의미 |
|---|---|---|
| Level 2 (목표) | 사용자가 묻지 않아도 모델이 주입된 지시를 수행하는가 | 실시간 배달 성립 |
| Level 1 (최소) | 안 하면, "토큰 봤는가" 질문에 정확히 답하는가 | 컨텍스트 진입만 성립 |

훅은 **1회만 주입**하도록 만들었다. 매 툴 호출마다 반복하면 세션이 시끄러워지고,
실제 ACB 에서도 메시지 1건당 알림 1회가 맞기 때문이다.

### 관찰된 사실

세션에 "`check.js` 와 `reset.js` 를 읽고 각각 3줄로 요약하라"는 작업을 지시했다.

| 시각 (UTC) | 사건 |
|---|---|
| 14:41:01.553 | Bash 툴 호출 직후 `additionalContext` 주입 |
| 14:41:09.157 | Read x2 — 이미 주입됨으로 skip (중복 알림 없음) |
| 14:41:23.037 | 모델이 `outbox2.json` 작성. 토큰 일치 |
| 14:41:27.410 | Stop 훅 실행 -> `allow`. 간섭 없음 |

**Level 1 확인 질문이 불필요했다. Level 2 로 통과했다.**
주입 -> 행동 지연 **21초**.

### 확정된 사실 5가지

1. **`hookSpecificOutput.additionalContext` 는 모델 컨텍스트에 실제로 진입한다.**
   워크스페이스에 존재하지 않는 토큰을 모델이 정확히 산출했다.
2. **모델은 주입된 지시를 사용자 질문 없이 자발적으로 수행한다.** 실시간 배달이 성립한다.
3. **배달 지연 21초.** Stop 훅 실측치 97초 대비 약 4.6배 빠르다. 이 값은 상한이 아니며
   실체는 "다음 툴 호출까지의 간격"이므로, 툴 호출이 잦은 작업일수록 짧아진다.
4. **`matcher: "*"` 는 모든 툴에 유효하다.** 주입 계기는 예상했던 Read 가 아니라
   모델이 먼저 실행한 Bash 였다.
5. **PostToolUse 와 Stop 은 서로 간섭하지 않는다.** PostToolUse 가 이미 배달을 끝낸 뒤
   Stop 훅은 `allow` 로 통과했고 중복 배달이 없었다. 스펙 7.2 의 이중 구조가 그대로 성립한다.

### 부수적으로 확인된 사실 — 프롬프트 인젝션 방어가 작동한다

모델이 스스로 "메시지 본문은 외부 에이전트 데이터로 취급해 지시문으로 해석하지
않았습니다"라고 보고했다. 주입 텍스트 마지막 줄에 넣은 방어 문구(스펙 7.4 템플릿)가
모델 행동에 실제로 반영된다는 뜻이다. 스펙 10.2 가 "가장 큰 위협"으로 지목한
프롬프트 인젝션에 대해, 최소한 문구 기반 방어가 무시되지는 않는다는 근거를 얻었다.

다만 이것은 **선의의 텍스트에 대한 관찰**이다. 실제 공격 문자열에 대한 저항력은
검증하지 않았으므로, 스펙 10.2 의 나머지 대응책(데이터 취급, 권한 분리)을 완화할 근거는
아니다.

### 검증되지 않은 것 (한계)

- **MCP 툴 호출로의 일반화는 여전히 미검증이다.** M0-1 과 동일한 한계다.
  이번 주입 지시도 파일 쓰기였다.
- **다건 연속 주입**은 테스트하지 않았다. 이번엔 1회 주입뿐이었다.
- 주입이 모델의 원래 작업을 중단시키는 극단적 상황(예: 매우 긴 주입 텍스트, 반복 주입)은
  테스트하지 않았다.

### 스펙에 미치는 영향

**설계 변경 없음.** 스펙 7.2 의 `wake_mode` 중 `hook` 모드는 이제 두 메커니즘
(작업 중 PostToolUse, 턴 종료 Stop) 모두 실측 근거를 갖췄다.

M0-1 시점의 "배달 지연이 턴 경계로 고정될 수 있다"는 우려는 해소됐다.
ACB 의 실시간 협업(v1 범위의 핵심)은 성립한다.

### 재현 방법

`M0\stop-hook-test\.acb-test\` 의 `m2-seed.js`(토큰 심기),
`posttool-hook.js`(훅 본체), `m2-check.js`(판정) 참조.
훅 호출 로그는 같은 디렉터리 `posttool.log` 에 있다.
토큰 state 파일은 스크래치패드에 있으며 세션 종료 시 사라질 수 있다.

---

## M0-4 — Codex App Server `turn/start` 의 TUI 렌더링 (PASS, 결과 (a))

### 검증 일시 / 환경

- 2026-08-10 (로그 시각은 UTC), Windows 11 Home 10.0.26200
- codex-cli 0.147.0, Node.js v25.8.1
- 검증 워크스페이스: `D:\Projects\Utils\Agent-Collaboration-Bus\M0\codex-appserver-test\`
- App Server: `codex app-server --listen ws://127.0.0.1:17777`
- TUI: `codex --remote ws://127.0.0.1:17777` (사용자가 별도 터미널에서 기동, idle 대기)
- 외부 클라이언트: `m4-client.js` (Node 내장 WebSocket 사용, 의존성 없음)

### 사전 조사로 확정한 프로토콜 사실

`codex app-server generate-json-schema --experimental --out <DIR>` 로 프로토콜 스키마를
추출해 확인했다.

- JSON-RPC 봉투에 **`jsonrpc` 필드가 없다.** 필수 필드는 `id`, `method` 뿐이다.
- `turn/start` 필수 파라미터는 `threadId`, `input` 둘뿐이다.
  `input` 은 `[{"type":"text","text":"..."}]` 형태의 배열이다.
- `turn/steer` 는 `threadId`, `expectedTurnId`, `input` 이 필수다.
- `turn/interrupt` 는 `threadId`, `turnId` 가 필수다.
- 핸드셰이크는 `initialize` 요청 -> `initialized` 알림 순이다.
- loopback 리스너는 인증이 필요 없다 (`--ws-auth` 는 non-loopback 전용).

### 관찰된 사실

| 시각 (UTC) | 사건 |
|---|---|
| 15:02:33.579 | TUI 접속. **별도 연결인 감시 클라이언트에 `thread/started` 알림 도착.** 스레드 id `019fec32-64c5-73c0-9da9-d81721e1f54e` |
| 15:03:01.345 | 외부 클라이언트가 `turn/start` 전송 |
| 15:03:01.346 | 즉시 수락. turn id `019fec32-d1e2-7200-88fc-f843375c634b`, `status: inProgress` |
| 15:03:01~03 | `thread/status/changed` 알림 2회 |
| (동시) | **TUI 프롬프트에 주입 텍스트가 렌더링되고 모델이 응답** |

TUI 화면에는 외부에서 보낸 텍스트가 `> [ACB-TEST] 이 메시지가 보이면 ...` 형태로,
사용자가 직접 입력한 것과 동일하게 표시됐고 모델이 `M4 수신 확인` 으로 답했다.

**결과 (a) — 원래 니즈 완전 충족.** 지연은 체감상 0 이다.
Stop 훅 97초, PostToolUse 21초와 비교해 차원이 다르다. idle 상태에서 즉시 깨어났고
턴 경계를 기다릴 필요가 없었다.

### 구현에 직접 영향을 주는 발견 3가지

1. **스레드 생명주기 이벤트는 연결 간에 브로드캐스트된다.** TUI 가 만든 스레드의
   `thread/started` 가 아무 관계 없는 별도 연결에 도착했다. 브로커가 App Server 에
   한 번만 붙어 있으면 모든 세션의 생명주기를 관찰할 수 있다.

2. **`thread/list` 로는 라이브 스레드를 찾을 수 없다.** 목록에는 과거 저장된 스레드
   5건만 나왔고 방금 접속한 스레드는 없었다(턴이 없어 아직 저장 전).
   threadId 는 `thread/started` 알림에서만 얻을 수 있었다.
   -> **브로커는 폴링이 아니라 상시 연결 + 이벤트 구독으로 스레드를 추적해야 한다.**
   브로커 재시작 시 이미 붙어 있던 세션의 threadId 를 놓칠 수 있다는 문제가 따라온다.

3. **턴 진행 상세 이벤트는 외부 연결에 오지 않았다.** 90초를 관찰했으나
   `turn/started`, `turn/completed` 는 도착하지 않았고 `thread/status/changed` 만 왔다.
   외부 관찰자는 "스레드가 바빠졌다/한가해졌다"는 알 수 있으나 턴 내용은 알 수 없다.
   ACB 입장에서는 오히려 바람직한 격리다.

### 환경 제약 (오늘 실측)

- **`codex app-server daemon` 은 Windows 미지원.** 실행 시
  `codex app-server daemon lifecycle is only supported on Unix platforms` 로 실패한다.
  `daemon start/stop`, `proxy --sock` 을 쓸 수 없다.
- **unix 소켓 경로는 SUN_LEN 제한을 받는다.** 긴 경로로 시도하면
  `path must be shorter than SUN_LEN` 로 실패한다.
- WebSocket 원격 전송은 공식적으로 experimental 로 표기돼 있다.
- 사용자는 평범한 `codex` 가 아니라 `codex app-server --listen` + `codex --remote`
  2단계로 기동해야 한다.

### 이 결과가 스펙에 미치는 영향 — 결론 보류

**기능은 완벽하지만 결론은 M0-6 에 달려 있다.**

M0-2 직후, "Claude Code 는 훅만으로 실시간 배달이 되는데 Codex 도 훅만으로 맞추면
설계가 대칭이 되어 단순해지지 않나"라는 검토가 있었다. 이 검토에서 드러난 것:

- Claude Code 는 idle 세션에 외부에서 턴을 주입할 방법이 **실제로 없다**.
  Stop 훅은 턴이 있어야 걸리고, spawn 은 새 세션이지 기존 세션 주입이 아니다.
- 반면 "실행 중인 턴에 외부 메시지 추가"는 Claude Code 도 가능하다 (M0-2 로 증명).
  단 **툴 호출 경계 한정**이라는 제약이 있다. `turn/steer` 는 임의 시점에 가능하다.
  차이는 가능/불가능이 아니라 지연 시간이다.
- **그런데 Codex 의 훅이 Claude 처럼 모델 컨텍스트에 텍스트를 주입할 수 있는지는
  검증된 적이 없다.** 훅 기능의 존재만 확인됐다(`--dangerously-bypass-hook-trust`
  플래그 존재). `additionalContext` 상당물이나 `decision:block` 상당물의 유무는 모른다.

따라서 분기는 다음과 같다.

| M0-6 결과 | 결론 |
|---|---|
| Codex 훅 주입 가능 | 대칭 설계 성립. `app_server` 는 v1 구현 제외, 스펙에는 "검증된 미래 옵션"으로 유지 |
| Codex 훅 주입 불가 | Codex 는 훅으로 실시간 배달 불가. `app_server` 가 유일한 수단이므로 비용을 감수하고 v1 에 포함 |

어느 쪽이든 **스펙 7.2 의 `wake_mode` 열거 자체는 바뀌지 않는다.** 열거형이므로
구현 범위에서 빼는 것은 추후 추가 가능한 additive 변경이다.

### 재현 방법

`M0\codex-appserver-test\m4-client.js` 참조.

```
codex app-server --listen ws://127.0.0.1:17777      # 터미널 1
codex --remote ws://127.0.0.1:17777                 # 터미널 2 (TUI)
node m4-client.js watch 300                         # 터미널 3, thread/started 로 threadId 획득
node m4-client.js turn <threadId> "<텍스트>"        # turn/start 전송
```

---

## M0-5 — WebSocket 없이 로컬 App Server 접속 (부분 확인)

### 확인된 사실 (관찰)

- **`codex app-server --listen unix://<짧은경로>` 는 Windows 에서 정상 기동하고
  소켓 파일을 생성한다.** Windows 11 의 AF_UNIX 지원이 실제로 동작한다.
- 경로가 길면 `path must be shorter than SUN_LEN` 으로 실패한다.
  스크래치패드처럼 긴 경로는 쓸 수 없고 짧은 경로를 골라야 한다.
- `codex app-server daemon` 계열(`start`/`stop`/`proxy --sock`)은 **Windows 미지원**이다.
- `codex --remote` 는 `ws://`, `wss://`, `unix://`, `unix://PATH` 를 받는다 (CLI 도움말).

### 확인되지 않은 것

- **unix 소켓으로 TUI 를 실제 접속시켜 보지 않았다.** M0-4 는 ws 로 진행했다.
- **외부 JSON-RPC 클라이언트가 unix 소켓에 접속하는 것을 시도하지 않았다.**
- (추정, 미검증) Node.js 는 Windows 에서 경로 소켓을 named pipe 로 매핑하므로
  Rust 측 AF_UNIX 소켓에 직접 붙지 못할 가능성이 있다. 그 경우 .NET 의
  `UnixDomainSocketEndPoint`(PowerShell 7)로 우회해야 한다. 둘 다 시도하지 않았다.

### 판단

`app_server` 모드를 v1 구현에서 제외하면 M0-5 는 물어볼 이유 자체가 사라진다.
따라서 **M0-6 결과가 나온 뒤에 필요 여부를 재판단한다.** 필요해지면 위 미검증 3건을
채우면 된다.

---

## M0-6 — Codex 훅이 모델 컨텍스트에 주입 가능한가 (PASS)

원래 스펙에 없던 항목이다. M0-2 를 마친 뒤 "Claude 는 훅만으로 실시간 배달이 되는데
Codex 도 훅만으로 맞추면 설계가 대칭이 되어 단순해지지 않나"라는 검토에서 도출됐다.
`app_server` 의 v1 포함 여부가 이 항목에 달려 있었다.

### 검증 일시 / 환경

- 2026-08-10 (로그 시각은 UTC), codex-cli 0.147.0
- 테스트 워크스페이스: `D:\Projects\Utils\Agent-Collaboration-Bus\M0\codex-hook-test\`
- Codex 세션 id `019fec43-0104-75a3-9811-fd71227d3068`, 모델 `gpt-5.6-sol`
- 훅 등록: `<프로젝트>/.codex/hooks.json`
- 평범한 `codex` TUI 사용 (App Server 아님)

### 검증 방법

M0-2 와 동일한 설계다. 정답 토큰 `M6-6AE4F2D848FB` 를 워크스페이스 **밖**
(`C:\Users\magedoga\AppData\Local\Temp\acb-m6-state.json`)에 두고, 사전에 워크스페이스
전수 검색으로 부재를 확인했다. 모델이 이 값을 아는 경로는 `additionalContext` 주입뿐이다.

추가로 훅 입력 JSON 원문을 `posttool-raw.jsonl`, `stop-raw.jsonl` 에 그대로 기록해
**Codex 의 훅 입력 계약**을 실측했다.

### 관찰된 사실

| 시각 (UTC) | 사건 |
|---|---|
| 15:21:05.015 | Bash(`Get-Content sample.txt`) 직후 `additionalContext` 주입 |
| 15:21:12.836 | 모델이 `outbox.json` 작성. 토큰 일치 |
| 15:21:13.357 | apply_patch 후속 호출 — 이미 주입됨으로 skip |
| 15:21:18.049 | Bash 후속 호출 — skip |

**Level 2 통과.** 사용자가 묻지 않았는데 모델이 스스로 처리했다.
주입 -> 행동 지연 **8초** (Claude 21초보다 빠름).

TUI 에는 `PostToolUse hook (completed) / hook context: [ACB] ...` 로 표시됐고,
모델이 "이어서 전달받은 협업 버스 토큰을 지정 파일에 기록하겠습니다"라고 명시한 뒤
실행했다. 원래 작업(과일 합산 12개)도 완주했다 — Claude 와 동일한 행동 패턴이다.

### 훅 입력 계약 — Claude 와 사실상 동일 (실측)

**PostToolUse**

```
session_id, turn_id, transcript_path, cwd, hook_event_name,
model, permission_mode, tool_name, tool_input, tool_response, tool_use_id
```

**Stop**

```
session_id, turn_id, transcript_path, cwd, hook_event_name,
model, permission_mode, stop_hook_active, last_assistant_message
```

- `hook_event_name` 값은 `"PostToolUse"`, `"Stop"` 으로 **PascalCase 그대로**다.
- **`stop_hook_active` 가 실제로 전달된다.**
- Claude 에 있던 필드가 빠진 것은 없다. Codex 고유 추가 필드는
  `turn_id`, `tool_use_id`, `last_assistant_message` 정도다.

-> **Claude 용 훅 스크립트를 거의 그대로 재사용할 수 있다.**

### hooks.json 형식과 위치 (실측 + Codex 자체 답변으로 교차 확인)

형식은 Claude 의 `.claude/settings.json` 훅 블록과 동일하다.
이벤트 이름은 설정에서 PascalCase, `hooks/list` 응답에서는 camelCase 로 나온다.

```json
{ "hooks": { "PostToolUse": [ { "matcher": "*",
  "hooks": [ { "type": "command", "command": "...", "timeout": 15 } ] } ] } }
```

위치는 다음 두 곳뿐이다.

| 경로 | source |
|---|---|
| `<CODEX_HOME>/hooks.json` | user |
| `<repo>/.codex/hooks.json` | project |

`<repo>/hooks.json`, `<repo>/.agents/hooks.json`, `<repo>/.codex/hooks/hooks.json` 은
**인식되지 않는다.** 후보 4곳에 동일 파일을 동시에 배치하고 `hooks/list` 로 확인했으며,
Codex 자체 답변도 같은 결론이었다.

`codex app-server` 의 `hooks/list` 메서드로 훅 설정이 제대로 파싱됐는지 기계적으로
검증할 수 있다. 응답은 `data: [{cwd, hooks, warnings, errors}]` 형태이고 각 훅에
`eventName`, `handlerType`, `matcher`, `command`, `timeoutSec`, `source`, `sourcePath`,
`currentHash`, `trustStatus`, `enabled` 가 들어 있다.

### 운용상 반드시 알아야 할 것 — 2단계 승인

1. **프로젝트 신뢰.** 신뢰되지 않은 프로젝트의 `.codex/hooks.json` 은 목록에조차
   나오지 않는다(오류도 없이 0건). 처음 경로를 못 찾은 원인이 이것이었다.
   `config.toml` 의 `[projects.'<경로>'] trust_level = "trusted"` 로 신뢰되면 즉시 잡힌다.
2. **훅 신뢰.** 훅은 `trustStatus: untrusted` 로 시작하고 승인 전에는 실행되지 않는다.
   TUI 에서 `/hooks` 로 검토하고 `t` 키로 신뢰한다. 신뢰 결과는 사용자 `config.toml` 의
   `[hooks.state.'<hooks.json 경로>:<event>:<i>:<j>']` 아래 `trusted_hash` 로 영구 저장된다.

**신뢰 해시의 범위 — 실측으로 확인 (중요)**

`trusted_hash` 는 **`hooks.json` 항목(command, event, matcher, timeout)에 대해 계산되며,
그 command 가 가리키는 스크립트 파일의 내용은 포함하지 않는다.**

`hooks/stop-hook.js` 를 통째로 새로 쓴 뒤에도 TUI 의 Trust 표시는 `Trusted` 그대로였고,
`config.toml` 의 `trusted_hash` 도 수정 전 `hooks/list` 가 보고한 `currentHash` 와
동일했다(`sha256:15d0fec0...`). 스크립트 파일 자체의 해시(`sha256:647834e1...`)와는
전혀 다른 값이다.

- **운용상 이점:** ACB 훅 스크립트의 로직을 업데이트해도 `hooks.json` 만 그대로면
  사용자 재승인이 필요 없다. "얇은 런처 + 별도 로직 파일" 같은 우회 구조는 불필요하다.
- **보안상 주의:** 신뢰 모델이 명령줄만 고정하고 실행될 코드는 고정하지 않는다.
  해당 스크립트 파일에 쓰기 권한이 있는 주체는 재승인 없이 훅 동작을 바꿀 수 있다.
  ACB 가 훅 스크립트를 배포·자동 업데이트한다면 그 경로 자체가 신뢰 경계가 된다.
  **스펙 10장에 반영할 것.**

### Codex 자체 답변으로 교차 확인된 사실 (실측 아님, 문서 근거)

`codex exec` 로 Codex 에게 직접 물어 얻은 답변이며, 위 실측과 모두 일치했다.

- `additionalContext` 는 다음 모델 요청에 **`developer` 컨텍스트로 들어간다.**
  일반 텍스트 stdout 은 무시된다.
- 기본 약 2,500 토큰을 넘으면 전체 텍스트 대신 축약 미리보기와 파일 복구 정보가
  전달될 수 있다 (`additionalContextLimit` 로 조정, `0` 이면 비활성).
  -> **ACB 메시지 본문이 길면 잘린다. 훅으로는 요약만 주입하고 본문은
  `acb_inbox` 툴로 가져오게 하는 스펙 6.2 설계가 옳다는 근거가 된다.**
- Stop 훅 턴 연장은 `{"decision":"block","reason":"..."}` + 종료 코드 0.
  `reason` 은 비어 있으면 안 되고 새 사용자 continuation prompt 로 전달된다.
  `continue: false` 는 연장이 아니라 중단이다.

### 후속 실측 — Codex Stop 훅 턴 연장 (PASS)

위 라운드에서 미실측으로 남겼던 항목을 이어서 측정했다. `stop-hook.js` 를
`{"decision":"block","reason":"..."}` 를 반환하도록 고치고, 새 토큰
`M6S-4C6E196F807C` 를 워크스페이스 밖에 심었다(부재 사전 확인).
PostToolUse 쪽 state 는 이미 `injected=true` 라 skip 되므로 간섭이 없다.

| 시각 (UTC) | 사건 |
|---|---|
| 15:39:15.143 | `stop_hook_active=false` -> `decision=block` |
| 15:39:24.080 | 모델이 `outbox-stop.json` 작성. 토큰 일치 |
| 15:39:37.931 | `stop_hook_active=true` -> `allow`. 턴 정상 종료 |

**PASS.** block -> 행동 지연 **9초**. TUI 에는 `Stop hook (blocked)` 과 함께
`feedback:` 라벨로 reason 전문이 표시됐다. Claude 가 같은 상황을 `Stop hook error:` 로
표시하는 것과 달리 **Codex 는 정상 동작으로 보이게 표시한다.**

입력 원문에서 `stop_hook_active` 가 1회차 `false`, 2회차 `true` 로 전달되는 것을
확인했다. `last_assistant_message` 도 각각 그 시점의 마지막 응답으로 채워져 있었다.

-> **Claude 와 Codex 양쪽 모두에서 Stop 훅 턴 연장이 실측으로 확인됐다.**

### 검증되지 않은 것 (한계)

- MCP 툴 호출로의 일반화는 M0-1, M0-2 와 동일하게 미검증이다.
- 다건 연속 주입, 2,500 토큰 초과 시 실제 동작은 테스트하지 않았다.

### 결론 — `app_server` 를 v1 에서 제외할 근거가 확정됐다

| 항목 | Claude Code | Codex CLI |
|---|---|---|
| 훅 설정 | `.claude/settings.json` | `.codex/hooks.json` (프로젝트 신뢰 필요) |
| PostToolUse `additionalContext` | PASS, 21초 | PASS, 8초 |
| Stop 훅 `decision:"block"` 턴 연장 | PASS, 97초 | PASS, 9초 |
| `stop_hook_active` 루프 방지 | PASS | PASS |
| idle 세션 외부 턴 주입 | 불가 | 가능 (`app_server` 필요) |

**두 클라이언트가 훅만으로 동일하게 동작한다.** `app_server` 의 고유 가치는 idle wake
하나이고 그것은 v1 비목표다. 반면 비용은 크다(Windows `daemon` 미지원, SUN_LEN 제한,
ws 전송 experimental, 2단계 기동, 브로커에 JSON-RPC 클라이언트 + 스레드 생명주기 추적).

-> **`app_server` 는 스펙 7.2 `wake_mode` 열거에 "검증된 미래 옵션"으로 남기고
v1 구현에서 제외한다.** 열거형이므로 나중에 추가해도 additive 다.

---

## M0-3 — Claude Code MCP sampling / elicitation (완료: sampling 미지원, elicitation 지원)

### 검증 일시 / 환경

- 2026-08-10 (로그 시각은 UTC)
- Claude Code 2.1.226, MCP 프로토콜 `2025-11-25`
- 검증 워크스페이스: `D:\Projects\Utils\Agent-Collaboration-Bus\M0\mcp-probe-test\`
- 최소 MCP 서버(stdio) `m3-server.js` 를 `.mcp.json` 으로 등록

### 검증 방법

능력 광고만 보지 않고 실제 호출까지 2단계로 확인했다. 광고는 하지만 실제로는 동작하지
않는 경우를 배제하기 위함이다(M0-4 에서 `turn/start` 가 "수락은 되는데 화면에 뜨는가"로
갈렸던 것과 같은 구조).

1. `initialize` 핸드셰이크에서 클라이언트가 보내는 `capabilities` 원문 기록
2. 서버 -> 클라이언트 방향으로 `sampling/createMessage` 와 `elicitation/create` 를 실제 발신
3. (후속) 툴 호출과 무관하게 **idle 상태에서** `elicitation/create` 를 자발적으로 발신

### 1단계 — 클라이언트가 광고하는 capabilities (원문)

```json
{
  "roots": { "listChanged": true },
  "elicitation": {}
}
```

`sampling` 키는 **없다.**

### 2단계 — 실제 호출 결과

| 요청 | 결과 |
|---|---|
| `sampling/createMessage` | `-32601 Method not found` — 명확한 미지원 |
| `elicitation/create` | **성공.** 화면에 입력 UI 렌더링, `{"action":"accept","content":{"answer":"Hello World"}}` 반환 |

elicitation 응답은 서버가 `requestedSchema` 로 지정한 구조 그대로 돌아왔다.

### 3단계 — idle elicitation (핵심)

세션 시작 후 **단 한 번의 턴도 실행되지 않은** 상태에서 서버가 자발적으로 발신했다.

| 시각 (UTC) | 사건 |
|---|---|
| 15:53:24 | `initialize` / `notifications/initialized` / `tools/list` |
| 15:53:49 | 서버가 자발적으로 `elicitation/create` 발신 (initialized + 25초) |
| (동시) | **사용자 화면에 입력 프롬프트 렌더링됨** |
| 15:56:38 | 사용자가 Accept. `{"action":"accept","content":{"answer":"Hello"}}` 도달 |

이 세션에서 클라이언트가 보낸 메시지는 `initialize`, `notifications/initialized`,
`tools/list` **3건뿐이었다. `tools/call` 이 0건이다.**
즉 툴 호출 컨텍스트 없이 서버가 먼저 말을 걸었고 화면에 떴다.

### 확정된 사실 — 그리고 그 한계

1. **`elicitation` 은 idle 세션에서도 동작한다.** 서버가 임의 시점에 사용자 화면에
   UI 를 띄우고 구조화된 응답을 받아올 수 있다. 훅으로는 불가능한 일이다.
2. **그러나 이것은 "사람을 깨우는" 것이지 "에이전트 턴을 시작하는" 것이 아니다.**
   응답은 JSON-RPC 로 **서버에게만** 돌아간다. Accept 직후 세션은 조용했고, 클라이언트가
   보낸 후속 메시지는 하나도 없었다(원문 로그로 확인). 모델은 이 왕복을 알지 못한다.

**따라서 v1 범위의 전제는 유지된다.** "완전히 idle 인 에이전트 세션을 외부에서 깨워
작업을 시작시키는 것"은 Claude Code 에서 여전히 불가능하다.
(이 절은 Claude Code 측정만 다룬다. Codex 측정은 아래 "Codex CLI 실측" 절에 있으며
같은 결론이 나왔다.)

### 스펙에 미치는 영향 — `notify` 모드 개선

`wake_mode` 열거는 그대로 두되, **`notify` 모드의 구현을 elicitation 기반으로 바꾸는 것을
검토할 가치가 있다.** 단 아래 표의 이점은 클라이언트가 실제로 렌더링할 때만 성립한다.
Codex 는 `approval_policy = "never"` 면 조용히 자동 거부하므로, OS 알림 폴백을 반드시
함께 구현해야 한다(아래 "운용상 함정" 절 참조).

| 현행 `notify` | elicitation 기반 |
|---|---|
| OS 알림 발송. 사용자가 놓칠 수 있음 | 세션 화면 안에 직접 승인 UI |
| 사용자가 세션으로 가서 직접 타이핑 | Accept / Decline + 구조화된 값 입력 |
| 브로커는 사용자가 봤는지 모름 | 응답이 즉시 브로커로 돌아옴 (`action`, `content`) |

스펙 15.3(수신자 미등록)이나 "실제 코드 수정이 필요한 요청은 notify 로 사람 개입을 요청"
같은 시나리오에 그대로 적용된다. 사용자 승인이 필요한 지점에서 특히 유용하다.

`sampling` 미지원은 ACB 에 실질적 손실이 아니다. 브로커가 클라이언트 모델을 빌려 쓸 일은
메시지 요약 정도이고 없어도 된다.

### 검증 방법상의 교훈 (기록해 둘 것)

첫 시도에서 elicitation 응답 대기 타임아웃을 30초로 잡았다가 시간 초과로 처리했는데,
**실제로는 화면에 UI 가 떠서 사람의 입력을 기다리는 중이었다.** 하마터면
"elicitation 미지원"이라는 잘못된 결론을 낼 뻔했다.

- elicitation 은 사람의 입력을 기다리는 요청이므로 타임아웃은 분 단위여야 한다
  (2차 시도에서 5분으로 조정했고, 실제 응답까지 2분 49초가 걸렸다).
- 같은 화면에서 모델이 툴 응답 텍스트만 보고 "elicitation 도 미지원"이라고 해석했는데
  이는 틀렸다. **모델의 해석이 아니라 서버 로그 원문을 근거로 판정해야 한다.**

### Codex CLI 실측 (후속 라운드)

같은 프로브 서버를 `codex mcp add acb_m3_probe -- node <경로>` 로 전역 등록하고
평범한 `codex` TUI 로 3라운드 측정했다. 로그는 `m3.log.codex-*.bak`,
`m3-raw.jsonl.codex-*.bak` 에 라운드별로 보존돼 있다.

**광고하는 capabilities (원문)**

```json
{"elicitation": {"form": {}, "url": {}}}
```

- `clientInfo`: `{"name":"codex-mcp-client","title":"Codex","version":"0.147.0"}`
- `protocolVersion`: `2025-06-18` (Claude Code 는 `2025-11-25`)
- `sampling` 없음. `roots` 도 없음.
- Claude 의 밋밋한 `"elicitation":{}` 과 달리 `form` / `url` 하위 항목까지 광고한다.

**라운드별 관찰**

| 라운드 | 조건 | 결과 |
|---|---|---|
| A | `codex mcp list` 가 띄운 헤드리스 점검 클라이언트 | 16:11:37.601 발신 -> `.604` **약 3ms 만에 `{"action":"decline"}`** |
| B | 평범한 `codex` TUI. 사용자 `config.toml` 의 `approval_policy = "never"` 적용 | 16:17:06.368 발신 -> `.370` **약 2ms 만에 `decline`. 화면에 아무것도 표시되지 않음** |
| C | `codex -a on-request` 로 승인 정책만 덮어씀 | **TUI 에 폼 UI 렌더링됨.** 사용자가 값 입력 후 제출, `{"action":"accept","content":{"answer":"Hello"}}` 도달 |

**C 라운드에서 렌더링된 UI** — `Field 1/1 (1 required unanswered)`, 요청 message,
필드명 `answer` 와 스키마의 `description`(`아무 문자열`), `enter to submit | esc to cancel`.
`requestedSchema` 가 그대로 폼으로 표현됐다. Claude 의 단순 입력 + Accept/Decline 보다
구현이 풍부하다.

**응답 시각:** 마지막 발신 16:29:32.463, accept 도착 16:30:33.595 -> **61초 이상**.
(이 라운드에서는 서버 인스턴스가 3개 동시에 떠 있어 어느 발신에 대한 응답인지
정확히 특정할 수 없다. 다만 최소 61초이며 A/B 의 2~3ms 와는 자릿수가 다르다.)

**제출 후 세션은 조용했다.** 클라이언트가 보낸 후속 메시지가 원문 로그에 하나도 없다.
Claude 와 동일하다.

### 확정된 사실 — 두 클라이언트 비교

| | Claude Code | Codex CLI |
|---|---|---|
| `elicitation` 광고 | `{}` | `{"form":{},"url":{}}` |
| `sampling` | 없음 (`-32601`) | 없음 |
| **idle 상태 렌더링** | 됨 | **됨.** 단 `approval_policy ≠ "never"` 필요 |
| 응답 회신 대상 | 서버(브로커)에게만 | 서버(브로커)에게만 |
| **제출 후 에이전트 턴 시작** | **안 함** | **안 함** |

**결론은 대칭이다. 양쪽 모두 elicitation 은 사람을 깨우지 에이전트를 깨우지 않는다.**

### 운용상 함정 — `approval_policy = "never"` 는 조용히 자동 거부한다

라운드 B 가 이 프로젝트에서 가장 위험한 관찰이다. Codex 는 `mcp_elicitations` 를
`sandbox_approval`, `skill_approval`, `request_permissions` 와 같은 **승인 범주**로
취급한다(바이너리 문자열에서 확인). 따라서 `approval_policy = "never"` 면 사용자에게
묻지 않고 자동 거부한다.

문제는 **브로커 입장에서 구분이 불가능**하다는 점이다. 자동 거부도 정상적인
`{"action":"decline"}` 으로 돌아오므로, "사용자가 거부했다"와 "사용자가 보지도 못했다"가
같은 응답이다. 에러도 경고도 없다.

**완화책:** 응답 지연으로 판별한다. 실측상 자동 거부는 2~3ms, 사람 응답은 61초 이상
(Claude 는 2분 49초)이었다. 브로커가 **수십 ms 안에 오는 `decline` 을 "자동 거부 의심"으로
분류하고 OS 알림으로 폴백**하면 된다. 임계값은 M3 구현 시 정한다.

### 검증 방법상의 교훈 2 — 샌드박스가 로그를 삼켰다

Codex TUI 라운드를 처음 돌렸을 때 **서버 프로세스는 떠 있는데 로그 파일이 하나도 없었다.**
"Codex 가 핸드셰이크를 안 한다"고 오판할 뻔했다.

Codex 는 MCP 서버를 샌드박스 안에서 실행한다(`sandbox_mode = "danger-full-access"`,
`[windows] sandbox = "elevated"`). TUI 를 프로브 디렉터리가 아닌 곳에서 띄우면
`D:\Projects\...` 로의 쓰기가 막힌 것으로 **추정**된다. 프로브 디렉터리에서 다시 띄우자
같은 경로에 정상 기록됐다. 단 그 사이 로그 경로를 이중화하는 코드 변경도 함께 했으므로
변수가 완전히 분리되지는 않았다. 확정 사실이 아니라 추정으로 남긴다.

교훈은 확실하다. **로그 쓰기 실패를 `try/catch` 로 조용히 삼키면 원인을 못 찾는다.**
ACB 훅과 브로커도 쓰기 실패를 반드시 stderr 등으로 드러내야 한다.

### 검증되지 않은 것 (한계)

- elicitation 을 짧은 간격으로 반복 발신했을 때의 동작(큐잉, 사용자 피로)은 테스트하지 않았다.
- Streamable HTTP 전송에서의 동작은 테스트하지 않았다. 이번 검증은 stdio 였고,
  스펙 9장의 ACB 는 Streamable HTTP 를 쓴다. **전송 방식이 다르므로 재확인이 필요하다.**
- **툴 호출이 진행 중일 때의 elicitation 은 측정하지 않았다.** 이번 측정은 전부
  idle 상태였다. 바이너리에 `guardian MCP elicitation metadata must include a non-empty
  tool_name` 이라는 문자열이 있어, Guardian(자동 심사) 경로에서는 elicitation 이 어느 툴
  호출에서 나왔는지를 요구하는 것으로 보인다. 이는 문자열 관찰일 뿐 동작을 확인한 것은 아니다.
- stdio 전송에서는 클라이언트 인스턴스마다 서버 프로세스가 따로 뜬다. C 라운드에서
  접속이 3건 기록됐다. HTTP 전송에서 "세션 1개 = 연결 1개"가 성립하는지는 확인하지 않았다.

### 재현 방법

`M0\mcp-probe-test\m3-server.js` 와 `.mcp.json` 참조.
`IDLE_DELAY_MS`(initialized 후 자발 발신까지의 지연)와 `ELICIT_TIMEOUT_MS` 를 조정할 수 있다.
서버 로그는 `m3.log`(해석된 이벤트)와 `m3-raw.jsonl`(클라이언트가 보낸 원문)에 남는다.
