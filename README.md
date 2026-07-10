# Publisher (매체사) 연동 가이드

단타게임(Service DT) 매체사용 연동 문서. 매체사가 게임 페이지로 사용자를 진입시키고, 콜백을 수신하고, 운영용 REST API 를 호출할 때 필요한 사양을 한 곳에 모은다.

| 항목 | 값 |
|---|---|
| 최초 작성일 | 2026-04-29 |
| 최종 개정일 | 2026-07-10 |
| 대상 | 매체사 개발자 / 인티그레이터 |

> **2026-07-10 개정 요약** — 콜백에 **재시도가 도입**되어 같은 이벤트가 두 번 이상 도착할 수 있다 (§3-5 멱등 처리 **필수**). 모든 베팅 콜백에 **`order_id`** 가 추가됐다. **스포츠(sports) 게임** 이 추가되어 베팅 선택지가 `up`/`down` 고정이 아니다 (§4-9). 상세는 §9 변경 이력.

---

## §1. 참여 신청 (Onboarding)

매체사 자가가입 채널은 없다. 영업/운영팀 합의 후 단타 운영팀이 매체 정보를 등록하고 자격증명을 발급한다.

### 1-1. 등록 시 매체사가 운영팀에 전달할 정보

| 항목 | 필수 | 설명 |
|---|---|---|
| 매체사 운영명 | ✓ | 매체사 명칭 (운영 식별용) |
| `client_id` | ✓ | 매체사 그룹 코드 (운영팀 합의로 결정) |
| `point_mode` | ✓ | `remote` = 매체사가 포인트 원장 보유 (베팅마다 승인 요청). `local` = 단타 서버가 포인트 관리 |
| `point_rate` | ✓ | 매체 포인트 환율 (예: 1 포인트 = 1000원이면 `1000`) |
| `state_callback_url` | 권장 | 라운드 이벤트 수신 URL — 미설정 시 라운드 콜백 미발송 |
| `bet_callback_url` | `remote` 시 ✓ | 베팅 승인/확정/취소 수신 URL |
| `event_callback_url` | 선택 | 단발성 이벤트(튜토리얼 보상) 수신 URL — 미설정 시 미발송 |
| `min_bet` / `max_bet` | 선택 | 매체별 베팅 한도 |
| `bet_options` | 선택 | 매체 UI 에 노출할 베팅 금액 프리셋 |
| `tutorial_enabled` / `tutorial_reward` | 선택 | 튜토리얼 보상 사용 여부와 기본 보상액 (§4-8) |

운영팀 검수 후 **`app_key`** 와 **`callback_secret`** 이 발급된다.

### 1-2. 발급 자격증명

- **`app_key`** — 모든 요청에 첨부 (헤더/쿼리/바디 중 하나). 외부 노출 가능 (단순 식별자).
- **`callback_secret`** — HMAC 서명에 사용. **외부 노출 금지**.

### 1-3. 환경

| 환경 | 도메인 |
|---|---|
| 개발 (dev) | `https://dantadev.snapplay.io/` |
| 운영 (live) | 대기중… |

---

## §2. 사용자 진입

매체사는 자사 사이트에서 단타 게임 페이지로 **링크** 한다. 진입 URL 쿼리 파라미터로 사용자 식별 정보를 전달한다.

### 2-1. 진입 URL 형식

```
https://dantadev.snapplay.io/?user_id={USER_ID}&adid={ADID}&app_key={APP_KEY}
```

### 2-2. 쿼리 파라미터

| 파라미터 | 별칭 | 필수 | 설명 |
|---|---|---|---|
| `user_id` | `uid` | ✓ | 매체사가 부여한 유저 식별자 |
| `app_key` | `appKey`, `appkey` | ✓ | 매체사 인증 키 |
| `adid` | — | 선택 | 광고 식별자 (없으면 빈 문자열) |

---

## §3. 콜백 (Callback) 수신

단타 서버는 이벤트마다 매체사의 콜백 URL 로 HTTP POST 를 발송한다. **본 절은 모든 콜백에 공통**이며, 이벤트별 페이로드 상세는 §4 에서 다룬다.

### 3-1. 공통 사양

| 항목 | 값 |
|---|---|
| 메서드 | `POST` |
| Content-Type | `application/json` |
| 인코딩 | UTF-8 |
| 타임아웃 | `bet_request` **5초** / 그 외 **10초** |

### 3-2. 공통 헤더

| 헤더 | 값 | 설명 |
|---|---|---|
| `X-App-Key` | `{app_key}` | 매체 식별 |
| `X-Signature` | `sha256={hex}` | HMAC-SHA256(`callback_secret`, raw_body) |
| `Content-Type` | `application/json` | |

### 3-3. 공통 바디 형식

```json
{
  "event_type": "round_started",
  "app_key": "your_app_key",
  "data": { ... },
  "timestamp": "2026-07-10T11:43:29.000Z"
}
```

`event_type` / `app_key` / `timestamp` 는 봉투에만 있고 `data` 안에 중복되지 않는다.

### 3-4. 서명 검증 (매체사 측 의무)

서명 대상은 **수신한 raw body 문자열 그대로** 다. JSON 파싱 후 재직렬화하면 검증에 실패한다.

```js
// Node.js 예시
const crypto = require('crypto');
function verify(rawBody, signatureHeader, callbackSecret) {
  const expected = 'sha256=' + crypto.createHmac('sha256', callbackSecret)
    .update(rawBody, 'utf8').digest('hex');
  return signatureHeader === expected;
}
```

### 3-5. 재시도와 멱등 처리 ⚠️

**같은 콜백이 두 번 이상 도착할 수 있다.** 매체사는 반드시 멱등 처리해야 한다.

| 이벤트 | 최대 시도 횟수 |
|---|---|
| `bet_confirmed`, `bet_canceled` | 3회 |
| `round_started`, `round_closed`, `round_ended`, `round_canceled`, `tutorial_reward` | 5회 |
| `bet_request` | 1회 (동기 호출, 재시도 없음) |

- 재시도 간격은 지수 백오프 — 2초, 4초, 8초, 16초 (최대 60초).
- **`2xx` 가 아닌 모든 응답은 실패로 간주되어 재시도된다** (4xx 포함). 이미 처리한 이벤트라도 `2xx` 로 응답할 것.
- 최대 시도 횟수를 넘기면 더 이상 발송하지 않는다. 이 경우 §6 REST API 폴링으로 보완한다.
- 재시도 외에도 내부 사정으로 중복 발송될 수 있다. **횟수와 무관하게 멱등성을 보장할 것.**

#### 멱등 키

| 이벤트 | 멱등 키 |
|---|---|
| `bet_request` / `bet_confirmed` / `bet_canceled` | **`order_id`** (58자, 예: `dt_202607_a3f7…`) |
| `round_ended` / `round_canceled` | `tr_id` |
| `tutorial_reward` | `completion_id` |

`order_id` 는 단타 서버가 발급하는 베팅 주문 식별자다. 하나의 베팅은 `bet_request` → `bet_confirmed` (또는 `bet_canceled`) 전 구간에서 **동일한 `order_id`** 를 갖는다. 매체사는 이 값으로 차감 row 를 찾아 중복 차감/환불을 방지한다.

> 마이그레이션 이전에 생성된 과거 베팅은 `bet_canceled` 의 `order_id` 가 `null` 일 수 있다. 이 경우 `bet_id` + `user_id` 로 매칭한다.

### 3-6. 응답 컨벤션

| HTTP 상태 | 의미 |
|---|---|
| `2xx` | 수신 성공 (처리 완료 또는 중복이라 무시함) |
| 그 외 | 수신 실패 → 재시도 |

> `bet_request` 만 응답 바디로 **승인 여부** 를 회신해야 한다. 다른 이벤트는 `2xx` = 수신 확인 의미.

### 3-7. 이벤트 타입 일람

| `event_type` | 채널 | 발송 시점 | 응답 의미 |
|---|---|---|---|
| `round_started` | `state_callback_url` | 라운드 OPEN (가격형 전용) | 수신 확인 |
| `round_closed` | `state_callback_url` | 베팅 마감 | 수신 확인 |
| `round_ended` | `state_callback_url` | 라운드 종료 + 정산 export 완료 | 수신 확인 |
| `round_canceled` | `state_callback_url` | 라운드 무효 처리 + 환불 export 완료 | 수신 확인 |
| `bet_request` | `bet_callback_url` | 베팅 시도 시 동기 호출 (`point_mode=remote` 만) | **승인/거부 응답 필수** |
| `bet_confirmed` | `bet_callback_url` | 베팅 트랜잭션 커밋 직후 | 수신 확인 (비동기 알림) |
| `bet_canceled` | `bet_callback_url` | 라운드/베팅 취소로 환불 필요 | 수신 확인 + **매체 측 `media_amount` 환불** |
| `tutorial_reward` | `event_callback_url` | 유저가 튜토리얼 완료 | 수신 확인 + **매체 측 `media_amount` 지급** |

> **`round_started` 는 가격형(coin 등) 게임에만 발송된다.** 스포츠/수동 운영 게임은 라운드 생성 주체가 달라 `round_started` 가 없고, `round_closed` 부터 수신한다.

> **취소 이벤트 (`round_canceled` / `bet_canceled`) 의 의미**: 라운드가 정상 종료되지 않고 무효(환불) 처리됐다는 알림이다. 매체사는 `bet_request` 단계에서 차감해놓은 `media_amount` 를 다시 자사 DB 에 환불해야 한다.

---

## §4. 콜백 페이로드 상세

이벤트별 `data` 필드 스키마. 모든 페이로드는 §3-3 의 공통 봉투 안에 들어간다.

`game_type` 은 게임 템플릿 코드다 (`coin`, `sports` 등 — 운영팀이 등록한 값).

### 4-1. `round_started` — 라운드 시작 (가격형 전용)

```json
{
  "event_type": "round_started",
  "app_key": "...",
  "data": {
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "1분 단위 BTC 가격 예측",
    "round_id": 7007,
    "round_number": 1160
  },
  "timestamp": "2026-07-10T11:43:29.000Z"
}
```

### 4-2. `round_closed` — 베팅 마감

**가격형** — `round_started` 와 동일한 `data` 스키마.

**스포츠** — 스키마가 다르다. `game_type` / `game_desc` 가 없고 경기 정보가 붙는다.

```json
{
  "event_type": "round_closed",
  "app_key": "...",
  "data": {
    "game_id": 21,
    "game_title": "프리미어리그 승부예측",
    "round_id": 8102,
    "round_number": 37,
    "fixture_id": 1035421,
    "kickoff_at": "2026-07-10T19:00:00.000Z",
    "teams": { "home": { "name": "Arsenal" }, "away": { "name": "Chelsea" } }
  },
  "timestamp": "2026-07-10T18:55:00.000Z"
}
```

### 4-3. `round_ended` — 라운드 종료 + S3 정산 export

라운드 정산이 끝나고 매체별 결과 JSON 이 S3 에 업로드된 직후 발송. **참여자 0명 매체에도 동일하게 발송**되어 매체가 능동적으로 "참여 없음" 을 확인할 수 있다.

**공통 필드**: `tr_id`, `game_id`, `game_title`, `game_type`, `round_id`, `round_number`, `result`, `start_price`, `end_price`, `user_count`, `s3_url`

```json
{
  "event_type": "round_ended",
  "app_key": "...",
  "data": {
    "tr_id": "f903236b80e7cfb0200044d68cd31c21fbd92153fd120598",
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "round_id": 7007,
    "round_number": 1160,
    "result": "up",
    "start_price": 113584000,
    "end_price": 113612500,
    "user_count": 12,
    "s3_url": "https://snap-service.s3.ap-northeast-2.amazonaws.com/danta/202607/games/9/rounds/7007/3.json"
  },
  "timestamp": "2026-07-10T11:48:00.000Z"
}
```

**스포츠 게임**은 위 공통 필드에 다음이 **추가**된다. 가격 개념이 없으므로 `start_price` / `end_price` 는 `null` 이다.

| 필드 | 설명 |
|---|---|
| `fixture_id` | 경기 식별자 |
| `result_direction` | 결과 선택지 코드 (`result` 와 동일 값) |
| `score` | 경기 스코어 객체 |
| `goals` | 득점 객체 |
| `teams` | `{ home, away }` |
| `settlement` | `{ winners, losers, totalPayout }` |

**수동 운영 게임**은 `result_direction`, `result_meta`, `settlement` 이 추가되며 강제 정산 시 `forced: true` 가 붙는다.

> S3 폴더의 `yyyymm` 부분은 **라운드 시작 시각의 KST(UTC+9) 기준** 이다 — 자정/월말을 걸쳐 종료되어도 시작 월 폴더에 일관 저장.

### 4-4. `round_canceled` — 라운드 무효 처리 + S3 환불 export

라운드 전체가 무효(환불) 처리되었음을 알리는 콜백. 채널은 `state_callback_url`. **참여자 0명 매체에도 발송**된다.

해당 라운드의 모든 베팅에 대해 별도로 `bet_canceled` 도 함께 발송되므로, 매체사는 보통 `bet_canceled` 로 단건 환불을 처리하고 `round_canceled` 는 라운드 단위 UI/통계 갱신에 사용한다.

```json
{
  "event_type": "round_canceled",
  "app_key": "...",
  "data": {
    "tr_id": "a4b1c8...",
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "round_id": 7007,
    "round_number": 1160,
    "reason": "engine_recovery_stale",
    "user_count": 12,
    "s3_url": "https://snap-service.s3.ap-northeast-2.amazonaws.com/danta/202607/games/9/rounds/7007/3.json"
  },
  "timestamp": "2026-07-10T11:48:00.000Z"
}
```

`reason` 값: `engine_recovery_stale`, `engine_recovery_orphan`, `inactive_game_cleanup`, `force_finish_failed`, `manual`

`round_ended` 와 달리 `result` / `start_price` / `end_price` 필드가 없다.

### 4-5. `bet_request` — 베팅 동기 승인 (`point_mode=remote`)

매체사가 포인트 원장을 보유하는 경우(`point_mode=remote`), 베팅이 발생할 때마다 **단타 서버가 매체 서버에게 동기 승인을 요청**한다. 채널은 `bet_callback_url`.

```json
{
  "event_type": "bet_request",
  "app_key": "...",
  "data": {
    "order_id": "dt_202607_a3f7b9c2e1d4f8a6b3c5e2d1f7a4b9c2e1d4f8a6b3c5e2d1",
    "request_ts": 1783252430000,
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160,
    "user_id": "u_42",
    "direction": "up",
    "amount": 1000000,
    "point_rate": 1000.0,
    "media_amount": 1000.0
  },
  "timestamp": "2026-07-10T11:43:50.000Z"
}
```

`request_ts` 는 요청 시각(ms) — 매체 측 replay 방지/감사용.

#### 매체사 응답 (필수)

```json
{ "approved": true }
```
또는 거부:
```json
{ "approved": false, "reason": "포인트 부족" }
```

- `approved` 가 `true` 가 아니면 단타 서버는 유저에게 **403** 으로 베팅을 거부하고, `reason` 문자열을 그대로 유저 화면에 노출한다.
- 승인 시 매체사는 이 시점에 자사 DB 에서 `media_amount` 만큼 포인트를 차감한다. 차감 row 에 **`order_id` 를 저장**해두어야 이후 `bet_confirmed` / `bet_canceled` 와 매칭할 수 있다.
- 응답 바디를 `{ "res_data": { "approved": true } }` 로 감싸도 승인으로 인정된다.

#### 매체사 응답이 없을 때

| 상황 | 결과 |
|---|---|
| 5초 타임아웃 / 네트워크 오류 / `2xx` 아님 | **베팅 거부** (승인으로 간주하지 않음) |
| `bet_callback_url` 연속 5회 실패 | 해당 URL 로의 요청을 **60초간 차단**, 그동안 베팅 전부 거부 |

콜백 엔드포인트 장애는 곧 매체 유저의 베팅 실패로 이어진다. 가용성을 확보할 것.

### 4-6. `bet_confirmed` — 베팅 확정 알림 (비동기)

단타 서버가 베팅 트랜잭션을 커밋한 직후 발송. 채널은 `bet_callback_url`. `bet_request` 페이로드에 `bet_id` 가 추가된 형태다.

```json
{
  "event_type": "bet_confirmed",
  "app_key": "...",
  "data": {
    "order_id": "dt_202607_a3f7…",
    "request_ts": 1783252430000,
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160,
    "user_id": "u_42",
    "bet_id": 9912,
    "direction": "up",
    "amount": 1000000,
    "point_rate": 1000.0,
    "media_amount": 1000.0
  },
  "timestamp": "2026-07-10T11:43:51.000Z"
}
```

### 4-7. `bet_canceled` — 베팅 환불 트리거 (비동기)

베팅이 무효 처리되어 매체사가 자사 DB 의 `media_amount` 를 환불해야 함을 알리는 콜백. 채널은 `bet_callback_url`.

발송 시점은 두 가지다.

**(a) race condition** — 매체사 승인 후 단타 서버 트랜잭션이 베팅 마감으로 실패. `reason: "round_closed"`.
이 경우 단타 서버에 베팅 INSERT 자체가 안 됐으므로 **`bet_id` 필드가 없다.** `order_id` 로 환불을 매칭한다.

**(b) 라운드 통째 취소** — `round_canceled` 와 함께 참여자별로 발송. `bet_id` 포함, `request_ts` 없음.
`reason` 은 §4-4 의 라운드 취소 사유와 동일.

```json
{
  "event_type": "bet_canceled",
  "app_key": "...",
  "data": {
    "order_id": "dt_202607_a3f7…",
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160,
    "user_id": "u_42",
    "bet_id": 9912,
    "direction": "up",
    "amount": 1000000,
    "point_rate": 1000.0,
    "media_amount": 1000.0,
    "reason": "engine_recovery_stale"
  },
  "timestamp": "2026-07-10T11:43:51.000Z"
}
```

> 매체사는 이 콜백 수신 시 `order_id` 로 차감 row 를 찾아 `media_amount` 를 환불한다. **이미 환불한 `order_id` 면 아무것도 하지 않고 `2xx` 응답** (§3-5).

### 4-8. `tutorial_reward` — 튜토리얼 보상 지급 (비동기)

유저가 튜토리얼을 완료했을 때 발송. 채널은 **`event_callback_url`** (미설정 시 미발송). 매체사는 `media_amount` 를 해당 유저에게 **지급(가산)** 한다.

보상은 매체별로 유저당 한 번만 지급된다. 단, 발송 자체는 재시도되므로 같은 이벤트가 여러 번 도착할 수 있다 — 멱등 키는 `completion_id`.

```json
{
  "event_type": "tutorial_reward",
  "app_key": "...",
  "data": {
    "client_id": "carrygold",
    "media_id": 3,
    "user_id": "u_42",
    "completion_id": 8812,
    "event_name": "튜토리얼 보상 지급",
    "game_id": -900,
    "round_id": -900,
    "amount": 16000,
    "media_amount": 16.0,
    "point_rate": 1000.0,
    "completed_at": "2026-07-10T11:40:00.000Z"
  },
  "timestamp": "2026-07-10T11:40:01.000Z"
}
```

> `game_id` / `round_id` 의 `-900` 은 "튜토리얼" 을 뜻하는 고정 마커다 (실제 게임 ID 는 항상 양수). `media_amount` 는 분수일 수 있으니 매체사가 자체 단위로 floor/round 처리한다.

### 4-9. `direction` 과 `result` 값 ⚠️ (스포츠 추가로 변경)

**`up` / `down` 고정이 아니다.** 라운드마다 허용 선택지가 정해진다.

| 게임 유형 | `direction` / `result` 값 |
|---|---|
| 가격형 (coin 등) | `up`, `down` — 결과는 `up` / `down` / `same` |
| 스포츠 | 라운드의 선택지 코드 (예: `home`, `away`, `draw`) |
| 수동 운영 | 운영자가 등록한 선택지 코드 |

라운드가 실제로 허용하는 선택지는 REST API 의 `currentRound.choices_snapshot` 에서 확인한다 (§6-5).

```json
"choices_snapshot": [
  { "code": "home", "label": "아스날 승", "meta": {} },
  { "code": "draw", "label": "무승부",   "meta": {} },
  { "code": "away", "label": "첼시 승",  "meta": {} }
]
```

매체사가 `direction` 을 자체 화면에 표시한다면, 코드값을 하드코딩하지 말고 `choices_snapshot` 의 `label` 을 사용할 것.

`result` 가 `same` 인 가격형 라운드는 **환불**이다 (시작가 == 종료가). 모든 베팅이 `bet_result: "draw"` 가 되며 `payout_amount == bet_amount`.

---

## §5. S3 정산 JSON

`round_ended` / `round_canceled` 의 `s3_url` 로 내려받는 매체별 정산 결과.

**경로 규칙** — `{prefix}/{YYYYMM(KST)}/games/{game_id}/rounds/{round_id}/{media_id}.json`
파일명은 `media_id` 이고, 폴더는 `game_id` 다.

```json
{
  "tr_id": "f903...",
  "generated_at": "2026-07-10T11:48:00.000Z",
  "game": {
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "..."
  },
  "round": {
    "round_id": 7007,
    "round_number": 1160,
    "result": "up",
    "result_reason": null,
    "result_reason_label": null,
    "start_price": 113584000,
    "end_price": 113612500,
    "started_at": "2026-07-10T11:43:00.000Z",
    "settled_at": "2026-07-10T11:48:00.000Z"
  },
  "media": {
    "media_id": 3,
    "app_key": "your_app_key",
    "point_rate": 1000.0
  },
  "summary": {
    "user_count": 12,
    "total_bet": 12000000,
    "total_payout": 22800000,
    "media_total_bet": 12000.0,
    "media_total_payout": 22800.0
  },
  "users": [
    {
      "order_id": "dt_202607_a3f7…",
      "bet_id": 9912,
      "user_id": "u_42",
      "direction": "up",
      "bet_amount": 1000000,
      "payout_amount": 1900000,
      "media_bet_amount": 1000.0,
      "media_payout_amount": 1900.0,
      "bet_result": "win"
    }
  ]
}
```

| 필드 | 단위 | 설명 |
|---|---|---|
| `bet_amount` / `payout_amount` / `total_bet` / `total_payout` | 원(KRW) | 단타 서버 내부 통화 |
| `media_bet_amount` / `media_payout_amount` / `media_total_*` | 매체 포인트 | `원 ÷ point_rate` 환산값 |
| `bet_result` | `win` / `lose` / `draw` | `draw` 는 환불 |

스포츠/수동 라운드는 `round` 객체 안에 §4-3 의 추가 필드(`fixture_id`, `score`, `teams` 등)가 함께 들어간다.

### 5-1. 취소된 라운드의 JSON

`round_canceled` 의 `s3_url` 은 같은 경로에 다음 차이를 갖는 JSON 을 올린다.

| 필드 | cancelled | settled |
|---|---|---|
| `status` | `"cancelled"` (최상위, 신규) | (없음) |
| `reason` | 취소 사유 (최상위, 신규) | (없음) |
| `round.result` | `"void"` | `up` / `down` / `same` / 선택지 코드 |
| `users[].bet_result` | 모두 `"draw"` | `win` / `lose` / `draw` |
| `users[].payout_amount` | `bet_amount` 와 동일 (전액 환불) | 결과별 산정 |
| `summary.total_payout` | `total_bet` 과 동일 | 결과별 산정 |

정상 종료인지 무효 처리인지는 **콜백의 `event_type`** 또는 **JSON 최상위 `status` 필드 유무**로 구분한다.

---

## §6. REST API

매체사 운영용 API. 두 prefix 모두 동일하게 동작한다.

- **권장**: `/api/pub/*`
- **호환 alias**: `/api/media/*`

### 6-1. 인증

모든 API 는 `app_key` 검증을 거친다. 아래 셋 중 **하나면 충분**.

| 우선순위 | 위치 | 키 |
|---|---|---|
| 1 | 바디 (POST) | `{ "app_key": "..." }` |
| 2 | 쿼리 | `?app_key={app_key}` |
| 3 | HTTP 헤더 | `X-App-Key: {app_key}` |

| 상태 코드 | 의미 |
|---|---|
| 401 | `app_key` 누락 또는 무효 |
| 403 | `is_active = false` 인 매체 |

### 6-2. 응답 컨벤션

성공:
```json
{ "result": 1, "data": { ... } }
```
실패:
```json
{ "result": 0, "message": "..." }
```

### 6-3. 엔드포인트 일람

| Method | Path | 설명 |
|---|---|---|
| `GET` | `/api/pub/me` | 인증된 매체사 본인 정보 (`callback_secret` 제외) |
| `GET` | `/api/pub/games/active` | 게임 리스트 |
| `GET` | `/api/pub/games/recent?limit=50` | 최근 종료된 라운드 (전체 게임) |
| `GET` | `/api/pub/games/:gameId?historyLimit=50` | 게임 상세 (현재 라운드 + 최근 회차) |
| `GET` | `/api/pub/games/:gameId/rounds?page=1&size=50` | 게임별 라운드 히스토리 (페이지네이션) |
| `GET` | `/api/pub/games/:gameId/rounds/:roundId` | 회차 단건 (S3 정산 메타 포함) |
| `GET` | `/api/pub/games/:gameId/users/:userId/bets?page=1&size=50` | 매체 본인의 특정 유저-게임 베팅 이력 |
| `GET` | `/api/pub/rounds/:roundId` | 회차 단건 (`gameId` 불필요) |

`limit` / `size` / `historyLimit` 은 기본 50, **최대 200** (초과 시 200 으로 절삭). `page` 는 1-based.

### 6-4. `GET /api/pub/me`

```json
{
  "result": 1,
  "data": {
    "media_id": 3,
    "media_name": "Carry Gold",
    "client_id": "carrygold",
    "app_key": "...",
    "point_rate": 1000.0,
    "point_mode": "remote",
    "state_callback_url": "https://your.domain/dt/state",
    "bet_callback_url":   "https://your.domain/dt/bet",
    "is_active": true
  }
}
```

### 6-5. `GET /api/pub/games/active`

> 이름과 달리 **진행중 게임만 반환하지 않는다.** 종료/비활성 게임도 함께 내려오므로, 매체사는 `status` 필드로 직접 필터링해야 한다.

주요 필드만 발췌 (실제 응답은 더 많은 운영 필드를 포함한다).

```json
{
  "result": 1,
  "data": [
    {
      "game_id": 9,
      "title": "비트코인 단타",
      "status": "active",
      "symbol": "KRW-BTC",
      "icon_url": "https://...",
      "template_code": "coin",
      "template_game_mode": "price",
      "round_duration": 300,
      "bet_cutoff": 30,
      "schedule_type": "24h",
      "market_open": true,
      "market_opens_at": null,
      "market_closes_at": null,
      "next_round_at": "2026-07-10T11:48:00.000Z",
      "currentRound": {
        "round_id": 7007,
        "round_number": 1160,
        "status": "open",
        "started_at": "2026-07-10T11:43:00.000Z",
        "start_price": null,
        "end_price": null,
        "result_direction": null,
        "total_players": 12,
        "total_bet_amount": 12000000,
        "choices_snapshot": [
          { "code": "up",   "label": "상승", "meta": {} },
          { "code": "down", "label": "하락", "meta": {} }
        ]
      }
    }
  ]
}
```

| 필드 | 설명 |
|---|---|
| `template_game_mode` | `price` / `sports` / `quiz` — 게임 유형 판정용 |
| `currentRound` | 현재 라운드. 없으면 `null` (스포츠 휴식기 등) |
| `currentRound.choices_snapshot` | 이 라운드가 허용하는 베팅 선택지 (§4-9) |
| `market_open` / `market_opens_at` / `market_closes_at` | 장 개장 상태 (주식/지수형) |

> 응답 키가 `currentRound` (camelCase) 임에 주의. 다른 필드는 snake_case 다.

### 6-6. `GET /api/pub/games/recent?limit=50`

```json
{
  "result": 1,
  "data": [
    {
      "round_id": 7006,
      "game_id": 9,
      "round_number": 1159,
      "status": "settled",
      "result_direction": "up",
      "result_reason": null,
      "result_reason_label": null,
      "start_price": 113584000,
      "end_price": 113612500,
      "started_at": "2026-07-10T11:38:00.000Z",
      "ended_at": "2026-07-10T11:43:00.000Z",
      "game_title": "비트코인 단타",
      "game_symbol": "KRW-BTC",
      "game_status": "active"
    }
  ]
}
```

### 6-7. `GET /api/pub/games/:gameId/rounds?page=1&size=50`

```json
{
  "result": 1,
  "data": {
    "game": { "game_id": 9, "title": "비트코인 단타", "template_code": "coin" },
    "page": 1,
    "size": 50,
    "total": 1160,
    "rounds": [ { "round_id": 7006, "round_number": 1159, "status": "settled", "…": "…" } ]
  }
}
```

### 6-8. `GET /api/pub/games/:gameId/rounds/:roundId`

응답에 **S3 정산 메타** (`settlement.s3_url` 등) 가 포함되어, 매체사가 콜백을 놓친 경우에도 동일한 데이터를 직접 가져올 수 있다.

```json
{
  "result": 1,
  "data": {
    "round_id": 7006,
    "round_number": 1159,
    "game_id": 9,
    "status": "settled",
    "result": "up",
    "result_reason": null,
    "result_reason_label": null,
    "start_price": 113584000,
    "end_price": 113612500,
    "started_at": "...",
    "ended_at": "...",
    "settled_at": "...",
    "point_rate": 1000.0,
    "settlement": {
      "tr_id": "f903...",
      "s3_url": "https://snap-service.s3.../202607/games/9/rounds/7006/3.json",
      "s3_key": "danta/202607/games/9/rounds/7006/3.json",
      "s3_bucket": "snap-service",
      "user_count": 12,
      "total_bet": 12000000,
      "total_payout": 22800000,
      "media_total_bet": 12000.0,
      "media_total_payout": 22800.0,
      "created_at": "..."
    },
    "results": [
      {
        "bet_id": 9912, "user_id": "u_42", "direction": "up",
        "bet_amount": 1000000, "payout_amount": 1900000, "bet_result": "win",
        "media_bet_amount": 1000.0, "media_payout_amount": 1900.0, "settled_at": "..."
      }
    ]
  }
}
```

> `settlement` 가 `null` 이면 아직 정산 export 가 완료되지 않은 상태.
> `results[]` 의 환산 필드명은 `media_payout_amount` 다 (`media_payout` 아님).

### 6-9. `GET /api/pub/games/:gameId/users/:userId/bets?page=1&size=50`

매체 본인이 등록한 유저(=`(client_id, app_key, user_id)`) 의 베팅만 노출한다. 다른 매체 데이터는 절대 노출되지 않는다.

---

## §7. 점검 체크리스트 (인티그레이션 완료 전)

- [ ] `app_key` / `callback_secret` 발급 완료, 안전한 위치에 저장
- [ ] HMAC-SHA256 서명 검증 로직 구현 — **raw body 문자열 기준** (모든 콜백)
- [ ] **모든 콜백 핸들러 멱등 처리** — `order_id` / `tr_id` / `completion_id` 기준 (§3-5)
- [ ] **이미 처리한 이벤트도 `2xx` 로 응답** (아니면 계속 재시도됨)
- [ ] `state_callback_url` — `round_started` / `round_closed` / `round_ended` / `round_canceled` 수신 처리
- [ ] `bet_callback_url` — `bet_request` 동기 승인 (`{approved}` 응답), **5초 내 응답**
- [ ] `bet_request` 승인 시 차감 row 에 **`order_id` 저장**
- [ ] `bet_confirmed` 수신 시 자사 DB 와 일치 확인
- [ ] **`bet_canceled` 수신 시 `order_id` 로 `media_amount` 환불** (`bet_id` 없을 수 있음)
- [ ] **`round_canceled` 수신 시 라운드 단위 UI/통계 갱신** (결과 표시 = "무효")
- [ ] `event_callback_url` 사용 시 `tutorial_reward` 수신 → `media_amount` **지급**
- [ ] **`direction` / `result` 를 `up`/`down` 으로 하드코딩하지 않기** — `choices_snapshot` 참조 (§4-9)
- [ ] 스포츠 게임은 `round_started` 가 오지 않음을 전제로 구현
- [ ] 매체사 사이트의 진입 링크에 `?user_id=...&app_key=...` 쿼리 파라미터 부착
- [ ] 콜백 endpoint 가 외부에서 접근 가능한지 (방화벽/whitelist) 확인
- [ ] 콜백 endpoint 가용성 확보 — 연속 실패 시 유저 베팅이 거부됨 (§4-5)
- [ ] 최대 재시도 초과 대비: §6 REST API 로 주기적 폴링
- [ ] S3 JSON 파싱: `users[]` + `summary` + **`status` 필드로 settled vs cancelled 구분**
- [ ] `result === 'same'` (환불) 케이스 매체 UI/포인트 처리

---

## §8. FAQ / 주의사항

**Q. `app_key` 가 노출되어도 되나요?**
A. URL 쿼리/HTTP 헤더로 평문 전달되므로 외부 노출 가능. 단순 식별자 역할이며, 인증의 핵심은 `callback_secret` 입니다.

**Q. 같은 콜백이 두 번 왔어요.**
A. 정상 동작입니다. 재시도 정책(§3-5)과 내부 사정으로 중복 발송될 수 있습니다. `order_id` / `tr_id` / `completion_id` 로 멱등 처리하고, 중복이면 아무것도 하지 않고 `2xx` 로 응답하세요.

**Q. 콜백 URL 이 응답 안 하면 어떻게 되나요?**
A. `bet_request` 는 5초 타임아웃 후 **베팅이 거부**됩니다. 그 외 이벤트는 10초 타임아웃 후 지수 백오프로 재시도되며(최대 3~5회), 모두 실패하면 발송이 중단됩니다. 누락분은 §6 REST API 폴링으로 보완하세요.

**Q. `bet_canceled` 에 `bet_id` 가 없어요.**
A. 매체사 승인 직후 베팅 마감으로 트랜잭션이 실패한 race condition 입니다 (`reason: "round_closed"`). 단타 서버에 베팅이 INSERT 되지 않아 `bet_id` 가 없습니다. `order_id` 로 환불을 매칭하세요.

**Q. `start_price` 가 `null` 이에요.**
A. 가격형 라운드가 `open` 단계(베팅 받는 중)이면 `start_price` 는 아직 캡처되지 않았습니다 (`closed` 시점에 확정). 스포츠 게임은 가격 개념이 없어 항상 `null` 입니다.

**Q. `result === 'same'` 의 의미는?**
A. 가격형에서 시작가와 종료가가 동일 → **환불 처리**. 모든 베팅이 `bet_result = 'draw'` 가 되며 `payout_amount === bet_amount`.

**Q. 스포츠 게임인데 `round_started` 가 안 와요.**
A. 정상입니다. 스포츠/수동 운영 게임은 라운드 생성 주체가 달라 `round_started` 를 발송하지 않습니다. `round_closed` 부터 수신합니다.

---

## §9. 변경 이력

| 일자 | 내용 |
|---|---|
| 2026-04-29 | 초기 발행 |
| 2026-07-10 | **재시도/중복 발송 도입** — "1회만 발송" 폐기, 멱등 처리 필수 (§3-5). **`order_id` 추가** — 모든 베팅 콜백의 멱등 키 (§3-5, §4-5~4-7). **스포츠 게임 반영** — `direction`/`result` 가 선택지 코드로 확장, `round_started` 미발송, `round_closed`/`round_ended` 페이로드 차이 (§4-2, §4-3, §4-9). **`tutorial_reward` 이벤트 신설** — `event_callback_url` 채널 (§4-8). **`point_mode` 명시** — `bet_request` 는 `remote` 에서만 발송 (§4-5). `bet_request` 타임아웃 5초로 명시, 실패 시 베팅 거부 + 연속 실패 시 60초 차단 (§4-5). **필드명 정정** — 콜백/S3 의 `game_code` → `game_id`, S3 파일명 `{app_key}.json` → `{media_id}.json`, `media_payout` → `media_payout_amount`. REST `current_round` → `currentRound`, `/games/active` 는 종료 게임도 반환. |
