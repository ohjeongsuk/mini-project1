# 조회 전용 챗봇 설계

> 작성 2026-09-17 · 상태 **승인됨** · 대상 저장소 셋 모두
> 이 문서는 구현 계획의 입력이다. 여기서 정하지 않은 것은 구현 중에 정하지 않는다.

---

## 1. 배경

`docs/PRD.md` 9장 2번에 **「LLM 소비 리포트(STAT-09)」**가 향후 확장으로 기록돼 있다.
이 문서가 정의하는 챗봇은 그 인접 기능이지만 성격이 다르다 — **리포트 생성이 아니라 대화형 조회**이고,
**LLM 을 쓰지 않는다.**

조회 대상 데이터는 이미 전부 존재한다. 새로 계산할 것이 없다.

| 필요한 것 | 이미 있는 엔드포인트 |
|---|---|
| 월 요약 · 카테고리별 · 예산 소진율 | `GET /api/v1/stats/monthly?yearMonth=&asOf=` |
| 최근 거래 내역 | `GET /api/v1/transactions?...` |

따라서 이 작업의 본체는 **「자연어 → 기존 API 파라미터」 변환 계층 하나**다.

---

## 2. 목표 / 비목표

### 목표

사용자가 아래 네 가지를 자연어로 물어 답을 받는다.

| 의도 | 예시 질문 |
|---|---|
| `MONTHLY_SUMMARY` | "이번달 얼마 썼어?" · "9월 수입 얼마야" |
| `CATEGORY_AMOUNT` | "지난달 식비 얼마 썼어?" · "교통 얼마" |
| `RECENT_TRANSACTIONS` | "최근 지출 보여줘" · "내역 10건 보여줘" |
| `BUDGET_STATUS` | "식비 예산 얼마 남았어?" · "예산 어때" |
| `FORECAST` | "이 속도면 얼마 쓸까?" · "이번달 예상 지출" |
| `RECURRING` | "고정지출 뭐 있어?" · "구독 결제 뭐 있어" |
| `DAILY_AMOUNT` | "어제 얼마 썼어?" · "15일 얼마 썼어" |

> **2단계 추가 (2026-09-17)** — 초안에서 미뤄뒀던 예측·고정지출·일별을 넣었다.
> 데이터가 이미 같은 응답 안에 있어(`forecast`·`daily`) 새 쿼리 없이 패턴과 문장만 늘었다.
> `RECURRING` 만 `stats/recurring` 을 따로 부른다.
>
> ⚠️ **의도 판정 순서가 이 추가의 핵심이다.** `"고정지출 뭐 있어"` 에는 `지출` 이,
> `"예상 지출 얼마야"` 에는 `지출`·`얼마` 가 들어 있어, 월 요약보다 **앞에** 두지 않으면
> 좁은 질문이 넓은 규칙에 먼저 잡힌다. 순서는
> 고정지출 → 예산 → 예측 → 일별 → 카테고리 → 최근내역 → 월요약 이다.
>
> ⚠️ **`RECURRING` 은 `yearMonth` 가 `null` 이다.** asOf 기준으로만 계산하므로
> 특정 달의 값처럼 보이면 안 된다.
>
> ⚠️ **날짜와 카테고리를 함께 물으면 한계를 밝힌다.** `daily` 배열에는 카테고리 구분이 없다.
> 카테고리 월 합계를 그날 것처럼 보여주는 것이 가장 나쁜 실패다.

### 비목표 (이번 범위 아님)

| 제외 | 이유 |
|---|---|
| LLM 연동 | 규칙 기반으로 확정. `PRD.md` 9장 2번에 별도 항목으로 남는다 |
| 멀티턴 문맥 ("그럼 지난달은?") | 이전 의도를 서버가 기억해야 하고, 그 순간 세션 저장이 필요해진다 |
| **대화 이력의 서버 저장** | 이력은 남기되 **브라우저**에 둔다(§6.1). 테이블·마이그레이션·조회 API 를 피한다 |
| **데이터 변경 (거래 추가·수정·삭제)** | **조회 전용.** 자연어 오해로 데이터가 바뀌면 되돌릴 방법이 없다 |
| 음성 입력 | 범위 밖 |

> ⚠️ **수입 카테고리별 집계는 서버에 아직 없다. 이번에 함께 추가한다.**
> `StatsService.monthly` 가 부르는 집계는 `sumExpenseByCategory` 이고, 응답의 `byCategory` 에는 **지출만** 들어 있다.
> `"급여 얼마야"` 에 `byCategory` 를 뒤져 답하면 **0원**이 나오는데, 이건 없는 값이 아니라 **틀린 값**이다.
>
> 대응은 §5.4 를 따른다 — 기존 쿼리의 하드코딩된 `EXPENSE` 를 파라미터로 바꾸고, 수입도 같은 쿼리로 조회한다.

---

## 3. 핵심 결정과 근거

### 3.1 규칙 기반 파서를 쓴다 (LLM 아님)

- 새 의존성 0, API 키 0, 호출 비용 0, 네트워크 의존 0.
- **동작을 단위 테스트로 고정할 수 있다.** LLM 응답은 매번 달라 같은 보장을 얻을 수 없다.
- `CLAUDE.md` §3 「임의로 라이브러리를 추가하지 않는다」와 충돌하지 않는다.
- 대가: 표현이 패턴을 벗어나면 못 알아듣는다. → `UNKNOWN` 응답에 **예시 질문을 함께 돌려주는 것**으로 처리한다.

### 3.2 파서는 백엔드에 둔다

프론트엔드에는 테스트 러너가 없다(`package.json` scripts = `dev`/`build`/`start`/`lint`).
분기가 많은 파서를 검증 수단이 없는 쪽에 두지 않는다. 백엔드에는 152건이 이미 돌고 있고,
`CsvParser`·`ForecastCalculator`·`RecurringDetector` 가 **DB 없는 순수 단위 테스트**라는 같은 패턴을 쓴다.

### 3.3 답변 문장도 서버가 만든다

`CLAUDE.md` §13: **"예측 수식은 백엔드에만 구현한다. 프론트가 같은 계산을 다시 하면 두 곳이 갈라진다."**
같은 원칙이 문장에도 적용된다. 문장을 프론트로 빼면 `"8월 식비는 412,000원이에요"` 가 맞는지 확인할 방법이 없어진다.

- 대가: 금액 천단위 포맷이 서버에도 생긴다(프론트 `lib/money.ts` 와 중복).
- 그 중복은 `ChatService` 안의 포맷 메서드 **한 곳**으로 한정한다. 규칙이 같으므로(ko-KR 천단위) 갈라질 여지가 작다.

### 3.4 `asOf` 를 클라이언트가 보낸다

`CLAUDE.md` §4 「"이번 달"과 "오늘"을 서버가 판정하지 않는다」를 그대로 따른다.
서버는 UTC 로 돌기 때문에 서버가 `now()` 로 "이번 달"을 정하면 **매월 1일 0~9시에 사용자는 한 달 전 답을 받는다.**
기존 `/stats/*` 와 동일하게 기준 날짜를 파라미터로 받는다.

**`ChatService`·`IntentParser` 에 `now()` 계열 호출이 등장하면 안 된다.**

### 3.5 해석 결과를 응답에 되돌려준다

`yearMonth` 를 응답에 담아 화면이 *"이렇게 알아들었습니다"* 를 표시할 수 있게 한다.
규칙 기반 파서는 오해할 수 있고, **오해했을 때 틀린 답을 맞는 답처럼 보여주는 것**이 가장 나쁜 실패 양상이다.

---

## 4. API 계약

> `CLAUDE.md` §2: API 계약이 바뀌면 **문서 저장소를 먼저** 수정한 뒤 백엔드 → 프론트엔드 순으로 반영한다.
> 따라서 아래 계약을 `CLAUDE.md` §5 에 먼저 반영한 뒤 구현을 시작한다.

### `POST /api/v1/chat`

인증 **필요**. 다른 모든 REST 응답과 같이 `ApiResponse<T>` 봉투를 쓴다.

#### 요청

```json
{ "message": "지난달 식비 얼마 썼어?", "asOf": "2026-09-17" }
```

| 필드 | 제약 |
|---|---|
| `message` | 필수, 공백만 있으면 안 됨(`@NotBlank`), 최대 200자 |
| `asOf` | 필수, `yyyy-MM-dd` |

`message` 상한을 200자로 두는 이유는 파서가 훑는 문자열의 길이를 제한하기 위해서다.
정상적인 질문은 30자를 넘지 않는다.

#### 응답

```json
{
  "success": true,
  "data": {
    "intent": "CATEGORY_AMOUNT",
    "answer": "8월 식비는 412,000원이에요. 전체 지출의 22%입니다.",
    "yearMonth": "2026-08",
    "transactions": null,
    "suggestions": []
  },
  "error": null
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `intent` | enum | `MONTHLY_SUMMARY` / `CATEGORY_AMOUNT` / `RECENT_TRANSACTIONS` / `BUDGET_STATUS` / `UNKNOWN` |
| `answer` | string | 화면에 그대로 표시할 완성 문장 |
| `yearMonth` | string \| null | 파서가 해석한 대상 월. `UNKNOWN` 이면 `null` |
| `transactions` | `TransactionResponse[]` \| null | `RECENT_TRANSACTIONS` 일 때만 채운다 |
| `suggestions` | string[] | `UNKNOWN` 일 때만 예시 질문 3개. 그 외 빈 배열 |

**`transactions` 는 기존 `TransactionResponse` 를 그대로 쓴다.** 새 DTO 를 만들면 프론트에 타입이 하나 더 생긴다.

#### 에러

| 상황 | 응답 |
|---|---|
| 토큰 없음 / 만료 | 401 `UNAUTHORIZED` (기존 `AuthenticationEntryPoint`) |
| `message` 공백 · 200자 초과 · `asOf` 형식 오류 | 400 `INVALID_INPUT` |
| **파서가 의도를 못 찾음** | **200 + `intent: "UNKNOWN"`** |

> ⚠️ **`UNKNOWN` 은 에러가 아니다.** 사용자가 예상 밖으로 물어본 것은 클라이언트 잘못도 서버 오류도 아니다.
> 4xx 로 만들면 프론트의 에러 처리 경로를 타서 토스트가 뜨고, 정작 보여줘야 할 예시 질문을 못 보여준다.

---

## 5. 백엔드 설계

### 5.1 파일

```
controller/ChatController.java        POST /api/v1/chat
dto/ChatRequest.java                  record (message, asOf)
dto/ChatResponse.java                 record (intent, answer, yearMonth, transactions, suggestions)
service/chat/IntentType.java          enum
service/chat/CategoryRef.java         record (name, type) — 파서에 넘길 카테고리 요약
service/chat/Intent.java              record (type, yearMonth, categoryName, categoryType, txnType, limit)
service/chat/IntentParser.java        ★ 순수 함수. 단위 테스트 대상
service/chat/ChatService.java         Intent -> 기존 서비스 호출 -> 문장 조립
```

`service/chat/` 하위 패키지를 두는 이유는 파일 넷이 하나의 관심사(자연어 해석)를 이루고,
기존 `service/` 가 이미 13개 파일을 담고 있기 때문이다.

### 5.2 `IntentParser` — 순수 함수

```java
// 카테고리는 사용자마다 다르므로 인자로 받는다. 이래야 파서가 순수해지고 DB 없이 테스트된다.
// 이름만이 아니라 type 까지 받는 이유는 수입 카테고리를 구분해야 하기 때문이다(§2 경고).
public static Intent parse(String message, LocalDate asOf, List<CategoryRef> categories)
```

**이 시그니처에 소유권 검증이 따라온다** — 넘기는 카테고리 목록이 인증 사용자의 것뿐이므로,
남의 카테고리 이름을 적어도 매칭되지 않는다. 별도 검증 코드가 필요 없다.

#### 기간 해석 (위에서부터 먼저 맞는 것을 쓴다)

| # | 패턴 | 결과 |
|---|---|---|
| 1 | `(\d{4})-(\d{1,2})` | 그 연·월 |
| 2 | `(\d{1,2})\s*월` | **언제나 asOf 의 연도**. 연도를 추측하지 않는다 |
| 3 | `지난\s*달` · `저번\s*달` · `전달` | asOf 월 − 1 |
| 4 | `이번\s*달` · `이달` · `금월` · `당월` | asOf 월 |
| 5 | (없음) | asOf 월 |

> **2번은 연도를 추측하지 않는다.** asOf 가 2026-09 일 때 `"12월"` 은 **2026-12** 이다.
> "미래니까 작년이겠지" 같은 보정을 넣으면, 사용자가 실제로 2026년 12월을 물었을 때
> 말없이 다른 달을 조회하고 그 사실이 화면에 드러나지 않는다. 추측해서 맞히는 것보다
> **곧이곧대로 해석하고 결과를 그대로 보여주는 편**이 낫다.
>
> 미래 달 조회는 안전하다. `ForecastCalculator.daysElapsed` 가 대상 월이 asOf 보다 뒤면 **0** 을 돌려주고,
> 이상치는 `daysElapsed < 7` 이면 계산하지 않으므로 0 으로 나누는 경로가 없다.
> 결과는 모두 0 이고 화면에는 빈 상태 문장(`{M}월에는 아직 기록이 없어요.`)이 나간다.
> 응답의 `yearMonth` 가 `2026-12` 로 내려가므로 사용자가 해석을 확인할 수 있다(§3.5).

#### 카테고리 해석

- `categories` 중 `message` 에 이름이 **부분 문자열로 포함된 것**을 찾는다.
- 여러 개가 맞으면 **가장 긴 이름**을 쓴다. `"주거/통신"` 과 `"통신"` 이 함께 있을 때 짧은 쪽이 먼저 잡히면 안 된다.
- 비교는 양쪽 모두 `toLowerCase()` 후 수행한다(영문 카테고리명 대비).
- 삭제된 카테고리는 대상에서 제외한다(카테고리 목록 조회 규칙과 동일).
- 맞은 카테고리의 `type` 을 `Intent.categoryType` 에 담는다. **이 값이 조회할 거래 종류를 정한다**(§5.4).

#### 거래 구분 해석 (`RECENT_TRANSACTIONS` 전용)

| 패턴 | `Intent.txnType` |
|---|---|
| `지출` · `쓴` · `썼` | `EXPENSE` |
| `수입` · `번` · `벌` | `INCOME` |
| (없음) | `null` = 전체 |

`"최근 지출 보여줘"` 가 수입까지 섞어 보여주면 질문에 답한 것이 아니다.

#### 의도 판정 (위에서부터 먼저 맞는 것)

| # | 조건 | 의도 |
|---|---|---|
| 1 | `예산` 포함 | `BUDGET_STATUS` |
| 2 | 카테고리가 매칭됨 | `CATEGORY_AMOUNT` |
| 3 | `내역` · `목록` · `보여` · `리스트` 포함 | `RECENT_TRANSACTIONS` |
| 4 | `얼마` · `지출` · `수입` · `썼` · `벌었` · `잔액` · `요약` · `수지` 포함 | `MONTHLY_SUMMARY` |
| 5 | 그 외 | `UNKNOWN` |

> **구체적인 것을 먼저 본다.** `"식비 예산 얼마 남았어"` 는 카테고리와 예산이 둘 다 있다.
> 예산을 먼저 보지 않으면 카테고리 조회로 새고, 사용자는 예산을 물었는데 지출액을 받는다.

#### 건수 해석 (`RECENT_TRANSACTIONS` 전용)

`(\d+)\s*건` 이 있으면 그 수, 없으면 **5**. **1~20 으로 클램프**한다.
상한이 없으면 `"1000건 보여줘"` 가 그대로 조회로 나간다.

### 5.3 `ChatService` — 조회와 문장 조립

**데이터를 새로 계산하지 않는다.** 기존 서비스를 그대로 호출한다.

| 의도 | 호출 |
|---|---|
| `MONTHLY_SUMMARY` · `BUDGET_STATUS` | `StatsService.monthly(userId, YearMonth, asOf)` 한 번 |
| `CATEGORY_AMOUNT` (지출) | `StatsService.monthly(...)` — `byCategory` 에 이미 들어 있다 |
| `CATEGORY_AMOUNT` (**수입**) | `StatsService.monthly(...)` + `sumByCategoryAndType(..., INCOME)` 한 번 더 (§5.4) |
| `RECENT_TRANSACTIONS` | 거래 목록 조회. `page=0`, `size=limit`, `type=txnType`, 정렬은 기본값 `txnDate,desc` + `id DESC` |
| `UNKNOWN` | **호출 없음** |

세 의도가 **같은 호출 하나로 처리되는 것**이 중요하다. `stats/monthly` 응답에 `summary`·`byCategory`·`budgets` 가
모두 들어 있어서다. 비율(`ratio`)도 이미 계산돼 있으므로 다시 나누지 않는다 — `BigDecimal.divide` 규칙을 피하는 부수 효과도 있다.

#### 답변 문장

`{M}` = 대상 월(숫자), 금액은 천단위 콤마 + `원`.

| 의도 | 정상 | 빈 상태 |
|---|---|---|
| `MONTHLY_SUMMARY` | `{M}월 지출은 {지출}원, 수입은 {수입}원이에요. 남은 돈은 {순액}원입니다.` | `{M}월에는 아직 기록이 없어요.` |
| `CATEGORY_AMOUNT` (지출) | `{M}월 {카테고리}는 {금액}원이에요. 전체 지출의 {비율}%입니다.` | `{M}월 {카테고리} 지출은 없어요.` |
| `CATEGORY_AMOUNT` (수입) | `{M}월 {카테고리}는 {금액}원이에요. 전체 수입의 {비율}%입니다.` | `{M}월 {카테고리} 수입은 없어요.` |
| `RECENT_TRANSACTIONS` | `최근 {n}건이에요.` (+ `transactions` 배열) | `아직 기록이 없어요.` |
| `BUDGET_STATUS` (카테고리 있음) | `{M}월 {카테고리} 예산 {예산}원 중 {사용}원을 썼어요. {남은}원 남았습니다.` | 예산 미설정: `{카테고리}에 {M}월 예산이 설정되어 있지 않아요.` |
| `BUDGET_STATUS` (카테고리 없음) | `{M}월 예산은 총 {합계}원 중 {사용}원을 썼어요.` | `{M}월에 설정된 예산이 없어요.` |
| `UNKNOWN` | `무슨 말씀인지 잘 모르겠어요. 이렇게 물어보실 수 있어요.` + `suggestions` | — |

예산 **초과** 시에는 `{남은}원 남았습니다` 대신 **`{초과}원 초과했어요.`** 를 쓴다.
음수를 "−188,000원 남았습니다"로 표시하면 읽는 사람이 한 번 더 계산해야 한다.

`suggestions` 고정 4개:
`"이번달 얼마 썼어?"` · `"지난달 식비 얼마 썼어?"` · `"최근 지출 보여줘"` · `"고정지출 뭐 있어?"`

프론트의 환영 카드도 **같은 목록**을 쓴다. 두 곳이 갈리면 사용자가 "못 알아들었어요" 화면에서
처음 보는 예시를 만난다.

#### 판정 기준 (해석의 여지를 남기지 않는다)

| 항목 | 규칙 |
|---|---|
| `MONTHLY_SUMMARY` 빈 상태 | `summary.income` 과 `summary.expense` 가 **둘 다 0** 일 때 |
| `CATEGORY_AMOUNT` 빈 상태 | 해당 카테고리가 결과에 **없거나** 금액이 0 일 때 |
| 비율 (지출) | `byCategory[].ratio` 를 그대로 쓰고 `× 100` 후 **정수 반올림**. 다시 나누지 않는다 |
| 비율 (수입) | `금액 ÷ summary.income` 을 **`divide(total, 4, RoundingMode.HALF_UP)`** 로 구한 뒤 `× 100` 정수 반올림 |
| 분모가 0 | `summary.income` 또는 `summary.expense` 가 0 이면 **비율 문장을 통째로 생략**한다. `NaN`·`Infinity` 가 나갈 경로를 만들지 않는다 |
| 금액 비교 | 전부 `BigDecimal.compareTo(...) == 0`. **`equals` 를 쓰지 않는다** (§4) |
| 금액 포맷 | `ChatService` 의 포맷 메서드 한 곳. `#,##0` + `원` |

### 5.4 기존 쿼리 하나를 파라미터화한다 (수입 카테고리 지원)

`StatsRepository.sumExpenseByCategory` 의 JPQL 은 거래 종류가 **하드코딩**돼 있다.

```java
and t.type = com.example.domain.TransactionType.EXPENSE   // 바꾼다
and t.type = :type                                        // 이렇게
```

| | 변경 |
|---|---|
| `StatsRepository` | `sumExpenseByCategory(userId, from, to)` → **`sumByCategoryAndType(userId, from, to, type)`** |
| `StatsService.monthly` | 호출부에 `TransactionType.EXPENSE` 를 넘긴다. **동작은 바뀌지 않는다** |
| `ChatService` | 수입 카테고리일 때 같은 메서드를 `INCOME` 으로 부른다 |

**쌍둥이 쿼리(`sumIncomeByCategory`)를 새로 만들지 않는다.** 12줄짜리 JPQL 이 두 벌이 되면
인덱스·`COALESCE`·삭제 카테고리 처리 같은 수정이 항상 한쪽에만 적용되는 날이 온다.
바뀌는 것이 상수 하나뿐이면 파라미터로 만드는 쪽이 맞다.

> ⚠️ **이 변경은 대시보드 경로를 지나간다.** `StatsService.monthly` 의 호출부가 바뀌므로,
> `StatsApiTest` 가 통과하는지로 회귀가 없음을 확인한다. 동작상 달라지는 것은 없어야 한다.

> **`sumExpenseByCategoryInPeriod`(이상치 기준선)는 건드리지 않는다.** 이상치는 지출에만 의미가 있고,
> 이번 작업이 그 경로를 쓰지 않는다. 관련 없는 리팩터링을 함께 하지 않는다(§13).

---

## 6. 프론트엔드 설계

```
src/app/(main)/chat/page.tsx          "use client" — 전체 화면
src/components/chat/ChatWidget.tsx    떠 있는 창 (FAB + 껍데기)
src/components/chat/ChatPanel.tsx     말풍선 목록 + 입력창 — 페이지와 위젯이 공유
src/components/chat/ChatMessage.tsx   말풍선 하나
src/hooks/useChat.ts                  useMutation
```

> **정정 (2026-09-17)** — 이 절의 초안은 "플로팅 패널은 얻는 것이 없다"며 페이지만 두기로 했다.
> **그 판단이 틀렸다.** 대시보드를 보면서 물어보는 것이 이 기능의 실제 쓸모인데,
> 페이지로만 두면 숫자를 확인하러 나갔다 와야 한다. 둘 다 둔다.

- **떠 있는 창(`ChatWidget`)과 전체 화면 페이지(`/chat`)를 함께 둔다.**
  대화 UI 는 `ChatPanel` 하나를 공유하고, 위젯은 껍데기(FAB·헤더·크기)만 맡는다.
  두 벌로 만들면 답변 렌더링이 갈라진다.
- **`/chat` 에서는 FAB 를 숨긴다.** 한 화면에 같은 입구가 둘일 이유가 없다.
- 위젯 크기: 데스크톱 `380×560`, 모바일은 `inset-x-4 top-20 bottom-36`.
  **모바일 FAB 는 `bottom-20` 이다** — 하단 탭 바가 64px 이라 `bottom-6` 이면 탭 위에 얹힌다.
- 그림자는 **`shadow-md`** 를 쓴다. §8 이 "그림자는 모달·드롭다운에만" 을 허용하고
  `popover`·`select` 가 이미 같은 값을 쓴다. `--hero-shadow` 는 잔액 카드 전용이며
  다크에서 `none` 이라 떠 있는 창에는 맞지 않는다.
- **FAB 에 배지를 달지 않는다.** 이 앱에 "안 읽음" 개념이 없어 항상 켜두면 거짓말이고, 끄면 장식만 남는다.
- **FAB 색은 액센트(`#4F46E5`)다.** `#EF4444` 는 이 앱에서 「지출」 전용 색이라 버튼에 쓰면 금액 색 체계가 무너진다.
- 대화 이력은 **localStorage 에 저장한다**(§6.1).
- `useChat` 은 `useMutation` 이다. 질문마다 새 요청이고 캐시할 대상이 아니다.
  조회 전용이므로 **`invalidateQueries` 를 호출하지 않는다.**
- 답변의 거래 목록은 **`TransactionRow` 를 재사용하지 않는다.** 그 컴포넌트는 `onDelete` 를 필수로 받고
  상세 링크를 감싸고 있어, 읽기 전용 세 줄을 위해 끌어오면 오히려 복잡해진다.
- 금액·날짜는 서버 문장을 그대로 쓴다. 거래 목록의 금액만 기존 `formatSignedAmount` 를 쓴다.
- `AppHeader` 의 데스크톱 네비와 모바일 하단 바에 항목 하나를 추가한다.
- `useReducedMotion` 을 존중한다(§8).

### 6.1 대화 이력 — localStorage

| 항목 | 값 |
|---|---|
| 키 | `moneylog_chat_{userId}` — `useAuth` 의 내 정보에서 얻는다 |
| 값 | 말풍선 배열(JSON). `role`·`content`·`intent`·`yearMonth`·`transactions` |
| 상한 | **최근 50개.** 넘으면 앞에서 버린다 |
| 로그아웃 | **지운다.** 기존 로그아웃 경로(토큰 제거 + React Query 캐시 비움)에 한 줄 추가 |

**키에 `userId` 를 넣는 이유** — 한 브라우저에서 다른 계정으로 로그인했을 때
이전 사용자의 대화가 보이면 안 된다. 로그아웃 시 지우는 것과 **이중으로** 막는다.

> ⚠️ **모든 읽기·쓰기를 `try/catch` 로 감싼다.** 프라이빗 모드·사이트 데이터 차단·용량 초과에서
> `localStorage` 접근은 **값을 못 주는 게 아니라 예외를 던진다.** 실패하면 조용히 메모리 상태로만 동작하고,
> 화면은 정상적으로 그려져야 한다.
>
> ⚠️ **첫 렌더에서 읽지 않는다.** 서버 HTML 에는 이력이 없으므로 렌더 시점에 읽으면 hydration 이 어긋난다.
> `ThemeToggle` 과 같은 방식으로 **마운트 이후**에 읽어 넣는다.
>
> ⚠️ **상한이 없으면 안 된다.** `transactions` 배열이 말풍선마다 최대 20건씩 붙으므로
> 무한 누적하면 localStorage 할당량(약 5MB)에 닿고, 그 순간 **쓰기가 예외로 바뀐다.**

### 상태 처리

| 상태 | 화면 |
|---|---|
| 전송 중 | 입력 비활성 + 답변 자리에 로딩 표시 |
| 오류(401 제외) | 말풍선에 "답을 가져오지 못했어요" + 재시도 버튼 |
| 401 | 기존 `apiClient` 가 처리 (토큰 폐기 후 `/login`) |
| 첫 진입 (이력 없음) | 예시 질문 3개를 버튼으로 노출. 누르면 그대로 전송 |
| 재진입 (이력 있음) | 저장된 말풍선을 그대로 복원하고 맨 아래로 스크롤 |

---

## 7. 테스트 (152 → 약 175건)

### `IntentParserTest` — 순수 단위 테스트

- 기간: `2026-08` / `8월` / `지난달` / `이번달` / 없음 → 각각 기대 `YearMonth`
- 기간: asOf 2026-09 에서 `"12월"` → **2026-12** (연도를 추측하지 않는다)
- 카테고리: 부분 일치 · **`"통신"` 과 `"주거/통신"` 중 긴 쪽** · 대소문자 무시 · 없는 이름
- 카테고리 type: `"급여"` → `categoryType = INCOME` 이 담긴다
- 의도 우선순위: `"식비 예산 얼마"` → `BUDGET_STATUS` (`CATEGORY_AMOUNT` 아님)
- 건수: `"10건"` → 10 · 없으면 5 · `"1000건"` → 20 클램프
- 거래 구분: `"최근 지출"` → `EXPENSE` · `"최근 내역"` → `null`
- `UNKNOWN`: 의도어 없음 · 인사말

### `ChatApiTest` — 통합 테스트

- 토큰 없이 호출 → 401 + `ApiResponse` 포맷
- 4종 의도 각각 정상 응답 (`answer` 에 기대 숫자가 들어 있는지)
- `message` 공백 → 400 `INVALID_INPUT`
- **남의 카테고리 이름을 넣어도 매칭되지 않는다** (소유권)
- 거래가 하나도 없는 달 → 500 이 아니라 빈 상태 문장
- **미래 달(`"12월"`, asOf 9월) → 500 이 아니라 빈 상태 문장, `yearMonth` 는 `2026-12`**
- **수입 카테고리(`"급여 얼마야"`) → 실제 수입 금액과 「전체 수입의 N%」**
- 수입이 0 인 달에 수입 카테고리를 물으면 → 비율 문장 없이 빈 상태 문장 (`NaN` 아님)
- `UNKNOWN` → **200** + `suggestions` 3개

### `StatsApiTest` — 회귀 확인 (기존 테스트, 새로 쓰지 않음)

§5.4 의 쿼리 파라미터화 후에도 대시보드 응답이 그대로인지 확인한다. **기존 테스트가 통과하면 된다.**

---

## 8. 작업 순서

| 단계 | 저장소 | 내용 |
|---|---|---|
| 1 | mini-project | `CLAUDE.md` §5 에 `POST /api/v1/chat` 계약 추가 |
| 2 | minipj1-backend | `sumByCategoryAndType` 파라미터화(§5.4) + `StatsApiTest` 회귀 확인 |
| 2 | minipj1-backend | `CategoryRef`·`Intent`·`IntentType` → `IntentParser` + 테스트 → `ChatService` → `ChatController` + 테스트 |
| 3 | minipj1-frontend | `types/api.ts` → `useChat` → `ChatMessage` → `ChatPanel`(이력 포함) → `page.tsx` → `AppHeader` → 로그아웃 시 이력 삭제 |
| 4 | — | 브라우저에서 4종 질문 실제 확인 |

각 단계는 별도 커밋이고 저장소를 섞지 않는다. 브랜치는 `feature/chatbot`.

---

## 9. 열린 결정

**없다.** 이 문서의 모든 항목이 구현 가능한 수준으로 확정되었다.
