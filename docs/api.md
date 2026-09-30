# API 레퍼런스와 운영 절차

> 백엔드 API 목록, 운영 절차, 세부 동작 메모다. 프로젝트 전체 개요는 [README](../README.md)에 있다.

용도별로 인증 방식이 다릅니다 — 손님용 API는 `merchantId`로 매장만 식별(사실상 무인증), POS 탭앱은
매장별 발급 토큰(`X-Store-Token`), 관리자 화면은 로그인 후 JWT, 배치 작업은 별도 공유 토큰을 쓰고,
정비 전산(ERP) 연동은 별도 공유 토큰(`X-ERP-Token`)을 씁니다.

## 손님용 (무인증, `merchantId`로 매장 식별)

| 메서드/경로 | 설명 | 인증 |
| --- | --- | --- |
| `POST /api/reservations` | 차량번호+정비항목(`serviceType`)+전화번호로 대기번호(매장·날짜별 독립 채번) 발급. `privacyConsent: true`가 없으면 400(개인정보 수집·이용 동의 필수), `marketingConsent: true`면 광고 수신동의 시각 기록. 응답에 `serviceDate`(접수일 KST) 포함, `peopleAhead`는 오늘 서비스 날짜 기준으로 계산. `Idempotency-Key` 헤더로 중복 접수 방지 | 없음 (10분당 5회 IP 레이트리밋) |
| `POST /api/payments` | 전화번호(+금액/`paymentKey`) 등록, 차량번호/정비항목은 같은 매장·전화번호의 예약 기록에서 자동 매칭. `privacyConsent: true` 필수(없으면 400), `amount`는 0~1억 사이 정수만 허용(그 외 400). 전자영수증 즉시 발송, 3개월 후 프로모션 예약(동의한 경우만) | 없음 (10분당 10회 IP 레이트리밋) |
| `POST /api/webhooks/toss/payment` | 결제 승인/취소 웹훅(백업 경로). `x-toss-webhook-id` 중복 방지 → 서명 검증(`x-toss-signature`) → 본 처리 순서로 실행하고, 본 처리가 실패하면 웹훅 기록을 지우고 500을 반환해 토스가 재시도하게 함. **`NODE_ENV=production`에서 `TOSS_WEBHOOK_SECRET` 미설정 시 서버가 부팅하지 않음**([`backend/.env.example`](../backend/.env.example) 참고) | 없음(웹훅 전용, 토스가 호출) |
| `GET /health` | 헬스체크(liveness). DB 접근 없이 즉시 `200 'ok'` | 없음 |
| `GET /health/ready` | 헬스체크(readiness). `SELECT 1` 성공 시 `200 {ok:true}`, 실패 시 `503 {ok:false}` — Cloud Run 등에서 새 리비전으로 트래픽을 넘기기 전 DB 연결 확인용 | 없음 |

## 관리자 화면용 (JWT)

목록 조회 4종(`GET /api/reservations`, `/api/reservations/failed`, `/api/payments`, `/api/payments/failed`)은
공통으로 아래 쿼리 파라미터와 응답 형태를 씁니다.

| 파라미터 | 기본값 | 설명 |
| --- | --- | --- |
| `storeId` | — | `hq_admin`만 유효(전체 매장 중 필터). `store_admin`이 보내면 무시하고 자기 매장으로 고정 |
| `date` | 없음(전체 기간) | KST 날짜 `YYYY-MM-DD`. 예약은 `serviceDate`, 결제는 `createdAt`의 KST 날짜 기준 |
| `status` | 없음 | 콤마로 구분한 상태값(예: `waiting,called`). `/failed` 계열은 이 값을 무시하고 각각 `notify_failed`/`receipt_failed`로 고정 |
| `q` | 없음(필터 미적용) | **신규.** 전화번호 또는 차량번호 부분일치 검색어. `GET /api/reservations`와 `GET /api/payments`에만 적용되고(`/failed` 계열은 미적용), 전화번호는 숫자만 남겨서 비교하므로 하이픈 포함 검색어도 매칭됨. 선행 와일드카드(`contains`) 검색이라 인덱스를 타지 않지만, 매장 단위 관리자 조회라 빈도가 낮아 허용 |
| `limit` | `100` | 1~500으로 클램프 |
| `offset` | `0` | 0 이상 |

응답은 `{ ok, count, total, hasMore, reservations }`(또는 `payments`) 형태입니다 — `count`는 이번 페이지
건수, `total`은 필터 적용 후 전체 건수, `hasMore`는 `offset + count < total`. **서버가 이미
`createdAt desc`로 정렬해서 내려주므로 관리자 화면에서 다시 정렬/역순 처리하지 않습니다.**

| 메서드/경로 | 설명 | 인증 |
| --- | --- | --- |
| `GET /api/reservations` | 예약 목록 조회(위 공통 파라미터/응답) | JWT (Bearer) |
| `GET /api/reservations/failed` | 순서 호출 실패(`status: 'notify_failed'`)와 접수(대기번호) 알림 실패(`intakeNotifyStatus: 'failed'`, 완료/취소 제외)를 **함께(union)** 조회. 응답 item에 `status`/`intakeNotifyStatus`가 모두 포함돼 있어 관리자 화면이 이 값으로 재발송 종류(`retry-notify`/`retry-intake`)를 구분함 | JWT (Bearer) |
| `GET /api/payments` | 결제 목록 조회(위 공통 파라미터/응답) | JWT (Bearer) |
| `GET /api/payments/failed` | 전자영수증 발송 실패 건(`receipt_failed`) 조회 | JWT (Bearer) |
| `GET /api/admin/summary` | **신규.** 지정 날짜(`date`, KST `YYYY-MM-DD`, 기본 오늘)·매장(`storeId`, 의미는 위 공통 파라미터와 동일) 기준 일별 요약. 예약 상태별 건수, 결제 건수·합계금액, 접수 알림 실패 건수(`intakeFailed`)를 한 번에 반환해 대시보드 상단 요약 카드에 씀(응답 형태는 §세부 동작 참고) | JWT (Bearer) |
| `POST /api/queue/call-next?storeId=` | 지정한 매장의 **오늘(KST) 서비스 날짜** `waiting` 중 대기번호가 가장 앞선 손님만 호출(어제 이월분은 호출 안 함), "순서입니다" 알림톡 발송 | JWT (Bearer) |
| `POST /api/reservations/:id/call` | 특정 예약을 순서와 무관하게 즉시 호출 (`waiting` 상태만 가능) | JWT (Bearer) |
| `POST /api/reservations/:id/complete` | 정비완료 처리. 이후 예약들의 "앞사람" 계산에서 빠짐 | JWT (Bearer) |
| `POST /api/reservations/:id/cancel` | **신규.** `waiting`/`called`/`notify_failed` → `cancelled`. 이미 취소된 건이면 `alreadyCancelled: true`로 200 응답, 이미 완료된 건이면 409 | JWT (Bearer) |
| `POST /api/reservations/:id/retry-notify` | **신규.** `notify_failed` 상태만 허용(아니면 409). 순서 안내 알림톡을 재발송하고 성공하면 `called`로, 실패해도 `notify_failed`를 유지한 채 200으로 응답(재시도 자체가 실패라는 신호는 응답의 `sent: false`로 전달) | JWT (Bearer) |
| `POST /api/reservations/:id/retry-intake` | **신규.** 접수 시 대기번호 안내 알림톡이 실패한 건(`intakeNotifyStatus: 'failed'`) 전용 재발송 — `completed`/`cancelled`가 아니어야 하며, 아니면 409. 앞사람 수(`peopleAhead`)를 최신값으로 다시 계산해 재발송하고, 성공하면 `intakeNotifyStatus`를 지우고(`null`) `sent: true`, 실패해도 `'failed'`를 유지한 채 `sent: false`로 200 응답(`retry-notify`와 동일한 패턴) | JWT (Bearer) |
| `DELETE /api/reservations/:id` | 예약 삭제 (테스트 데이터 정리) | JWT (Bearer) |
| `POST /api/payments/:id/retry-receipt` | **신규.** `receipt_failed` 상태만 허용(아니면 409). 전자영수증 알림톡 재발송, 응답 형태는 `retry-notify`와 동일한 패턴(`sent` 플래그) | JWT (Bearer) |
| `GET /api/admin/stores` | 등록된 가맹점(매장) 목록 조회. 각 매장 객체에 `posToken`(POS 탭앱 인증용) 포함 | JWT (Bearer), `hq_admin` 전용 |
| `POST /api/admin/stores` | 가맹점 등록. `merchantId`+매장명(+사업자번호)을 받아 내부 `store_id`와 **POS 토큰을 함께 자동 발급** | JWT (Bearer), `hq_admin` 전용 |
| `POST /api/admin/stores/bulk` | 매장 대량 등록(최대 500건), 항목별 성공/실패를 나눠서 반환. 성공 항목마다 `posToken` 포함 | JWT (Bearer), `hq_admin` 전용 |
| `POST /api/admin/stores/:id/pos-token` | **신규.** 해당 매장의 POS 토큰을 재발급(회전) — 기존 토큰은 즉시 무효화됨. 없는 매장이면 404 | JWT (Bearer), `hq_admin` 전용 |
| `POST /api/admin/stores/:id/erp-code` | **신규.** 해당 매장에 정비 전산 측 매장 코드(`erpStoreCode`)를 등록/해제([정비 전산(ERP) 연동](./erp-integration.md)). `1~64자`, `[A-Za-z0-9_-]+`만 허용(위반 400), 다른 매장이 이미 쓰는 코드면 409, 빈 문자열/`null`을 보내면 코드 해제 | JWT (Bearer), `hq_admin` 전용 |
| `POST /api/admin/store-admins` | 매장 관리자 계정 발급. `merchantId`로 매장을 찾아 그 매장에 스코프된 `store_admin` 계정 생성 | JWT (Bearer), `hq_admin` 전용 |
| `POST /api/admin/login` | 이메일/비밀번호로 로그인, JWT 발급(`role`: `hq_admin`\|`store_admin`, `storeId` 포함). 기존 IP 레이트리밋에 더해 **같은 계정 비밀번호 5회 연속 실패 시 15분 계정 잠금**(423 응답) 추가 | 없음 |
| `GET /api/admin/me` | 로그인한 관리자 본인 정보(역할, 소속 매장) 조회. `store` 객체에 `posToken` 포함(자기 매장이므로 `store_admin`도 조회 가능) | JWT (Bearer) |

## POS 탭앱용 (`X-Store-Token`, 인증 방식이 바뀜)

`merchantId`는 더 이상 POS API의 인증 수단이 아닙니다(요청에 실려도 무시됩니다) — 매장별로 발급된
64자리 hex 토큰을 `X-Store-Token` 헤더로 보내야 합니다. 토큰이 없거나 틀리면 401
(`STORE_TOKEN_REQUIRED`/`INVALID_STORE_TOKEN`), 매장이 비활성 상태면 403을 반환합니다. 발급/재발급
절차는 바로 아래 [POS 토큰 발급·재발급 운영 절차](#pos-토큰-발급재발급-운영-절차) 참고.

| 메서드/경로 | 설명 | 인증 |
| --- | --- | --- |
| `GET /api/pos/queue` | 오늘(KST) 서비스 날짜 접수분 전체 + 아직 안 끝난(취소·완료 아님) **이월 건**(전날 이하 `called`/`notify_failed`, 어제 `waiting`은 노쇼로 보고 제외)을 서비스 날짜순·대기번호순으로 조회. 각 item에 `serviceDate`가 포함돼 응답 최상위 `serviceDate`(오늘 날짜)와 다르면 이월 건임을 화면에서 구분할 수 있음. **전화번호는 마스킹**(`010-****-5678`)해서 내려줌(POS 화면에 원본이 필요 없음) | `X-Store-Token` |
| `POST /api/pos/queue/:id/call` | 오늘 등록된 해당 매장 예약만 호출 가능(아니면 404) — 이월 건은 이미 `called`라 어차피 호출 대상이 아니고, 어제 `waiting`을 오늘 실수로 새로 호출하는 사고를 막기 위해 **이 endpoint만 날짜 제한을 유지**함 | `X-Store-Token` |
| `POST /api/pos/queue/:id/complete` | 오늘 등록분은 물론 **이월 건도** 정비완료 처리 가능(날짜 제한 없음). 해당 매장 예약이 아니면 404 | `X-Store-Token` |
| `POST /api/pos/queue/:id/cancel` | **신규.** 노쇼 등으로 대기열에서 제외. 오늘 접수분·이월 건 모두 가능(날짜 제한 없음). `waiting`/`called`/`notify_failed` → `cancelled` | `X-Store-Token` |

## 정비 전산(ERP) 연동용 (`X-ERP-Token`)

**신규.** 자세한 흐름·토스 확인 사항은 [정비 전산(ERP) 연동](./erp-integration.md) 문서 참고. 현재 주 경로인 장바구니 방식(`/api/erp/carts`)의 계약은 [전산-POS-장바구니-연동설계](./전산-POS-장바구니-연동설계.md) 4절에 있다.

| 메서드/경로 | 설명 | 인증 |
| --- | --- | --- |
| `POST /api/erp/draft-orders` | 전산이 보낸 상품 목록(`storeCode`/`referenceId`/`items[]`/`totalAmount`/`memo`)으로 해당 매장 토스 POS에 미결제(OPENED) 주문을 생성. `referenceId`가 멱등키 — 같은 값으로 재호출하면 토스를 다시 부르지 않고 기존 결과를 `200 { duplicate: true }`로 반환. 검증 실패(필드 누락/형식 오류/`totalAmount`가 항목 합계와 불일치 등) 400, 미등록 `storeCode` 404, 비활성 매장 403, 토스 측 오류 502(토스 원본 에러 메시지는 클라이언트에 노출하지 않고 로그에만 남김 — 같은 `referenceId`로 재시도 가능) | `X-ERP-Token` |
| `GET /api/erp/draft-orders/:referenceId` | 전산이 앞서 보낸 주문의 처리 결과를 재조회. 없으면 404 | `X-ERP-Token` |

미설정(`ERP_API_TOKEN` 없음) 시 두 endpoint 모두 503을 반환합니다(환경 변수는 [`backend/.env.example`](../backend/.env.example) 참고).

## 내부 배치 작업용 (`X-Promotion-Job-Token`, Cloud Scheduler가 호출)

| 메서드/경로 | 설명 | 인증 |
| --- | --- | --- |
| `POST /internal/jobs/send-promotions` | 결제 3개월 경과 + 광고성 정보 수신에 동의(`marketingConsentAt` not null)한 손님에게 프로모션 알림톡 발송. 100건씩 배치로 claim해 **1회 실행 최대 `PROMO_MAX_PER_RUN`건(기본 2000) 또는 50초 예산**까지 처리하고, 상한에 걸려 남은 대상이 있으면 `exhausted: true` | `X-Promotion-Job-Token` |
| `POST /internal/jobs/purge-expired` | **신규.** `DATA_RETENTION_DAYS`(기본 1095일)보다 오래된 예약/결제 중 아직 파기 안 된 건의 전화번호·차량번호를 익명화(`anonymizedAt` 기록). 한 번에 최대 1000건씩 최대 10회 반복 | `X-Promotion-Job-Token` |

## Cloud Scheduler 잡 등록 절차

위 두 endpoint는 **코드가 이미 만들어져 있을 뿐**입니다 — 서버 안에는 더 이상 `node-cron` 같은 자체
스케줄러가 없으므로, **Cloud Scheduler에 잡을 등록해두지 않으면 두 작업 모두 영원히 자동으로 실행되지
않습니다**(수동으로 curl/Postman으로 호출하지 않는 한). 이 등록은 **운영 담당자가 직접** gcloud CLI(또는 GCP
콘솔의 Cloud Scheduler 화면)로 해야 하는 작업입니다 — 로컬에 `gcloud`가 없다면
[Cloud Shell](https://console.cloud.google.com/cloudshell)에서 브라우저로 바로 실행할 수 있습니다.

- **`send-promotions`** — 매일 10:00 KST (3개월 경과 프로모션 알림톡 발송)
- **`purge-expired`** — 매일 04:00 KST (개인정보 보관기간 경과분 파기, 트래픽이 적은 새벽 시간대로 지정)
- 대상 리전 `asia-northeast3`, 프로젝트 `<GCP_PROJECT_ID>`

```bash
gcloud scheduler jobs create http queue-send-promotions \
  --project=<GCP_PROJECT_ID> \
  --location=asia-northeast3 \
  --schedule="0 10 * * *" \
  --time-zone="Asia/Seoul" \
  --uri="https://<CLOUD_RUN_URL>/internal/jobs/send-promotions" \
  --http-method=POST \
  --headers="X-Promotion-Job-Token=<PROMOTION_JOB_TOKEN 실제 값>"

gcloud scheduler jobs create http queue-purge-expired \
  --project=<GCP_PROJECT_ID> \
  --location=asia-northeast3 \
  --schedule="0 4 * * *" \
  --time-zone="Asia/Seoul" \
  --uri="https://<CLOUD_RUN_URL>/internal/jobs/purge-expired" \
  --http-method=POST \
  --headers="X-Promotion-Job-Token=<PROMOTION_JOB_TOKEN 실제 값>"
```

`<PROMOTION_JOB_TOKEN 실제 값>`은 Secret Manager/Cloud Run 서비스에 설정된 실제 토큰 문자열로 바꿔서
실행하세요(이 문서나 커밋에 실제 토큰 값을 남기지 말 것). Cloud Run 서비스 자체는 공개 상태로 두고
이 헤더 하나로만 인증하는 구조이므로(`backend/.env.example`의 `PROMOTION_JOB_TOKEN` 설명 참고),
`--oidc-service-account-email` 같은 IAM 인증 플래그는 필요 없습니다.

**재시도는 안전합니다.** Cloud Scheduler가 타임아웃/네트워크 문제로 같은 실행을 다시 호출해도,
`send-promotions`는 대상을 먼저 claim한 뒤 `promoSent`로 표시하므로 이미 보낸 대상을 다시 잡지 않고,
`purge-expired`는 이미 `anonymizedAt`이 찍힌 건을 다시 대상으로 삼지 않습니다 — 그러므로 중복 발송·
중복 파기 걱정 없이 재시도 정책을 기본값(Cloud Scheduler 기본 재시도)으로 둬도 됩니다.

> `send-promotions`는 2026-08-05 GCP 초기 구성 때 `queue-promotion-daily`라는 이름으로 이미 한 번
> 등록됐을 수 있습니다([gcp-migration-and-scale-plan.md](./gcp-migration-and-scale-plan.md)의 실행 기록 참고). 이후 `PROMOTION_JOB_TOKEN`이
> 교체된 적이 있어 그 잡의 헤더가 최신 토큰인지 불확실합니다 — 새로 만들기 전에
> `gcloud scheduler jobs describe queue-promotion-daily --location=asia-northeast3 --project=<GCP_PROJECT_ID>`로
> 기존 잡이 있는지 먼저 확인하세요. 있다면 이름이 다른 잡을 새로 만들어 endpoint를 이중으로 호출하게
> 만들지 말고, 아래처럼 헤더만 최신 토큰으로 갱신하는 편이 안전합니다.
>
> ```bash
> gcloud scheduler jobs update http queue-promotion-daily \
>   --location=asia-northeast3 --project=<GCP_PROJECT_ID> \
>   --headers="X-Promotion-Job-Token=<PROMOTION_JOB_TOKEN 실제 값>"
> ```
>
> `purge-expired`는 이번 라운드 이전 v2 하드닝에서 endpoint만 추가된 상태라, Cloud Scheduler 잡 자체가
> 아직 없을 가능성이 높습니다 — 위 `create` 명령으로 새로 등록하면 됩니다.

## POS 토큰 발급·재발급 운영 절차

POS 탭앱은 이제 매장마다 다른 64자리 hex 토큰(`Store.posToken`)이 있어야 대기열 API를 호출할 수
있습니다(§API POS 탭앱용). 운영 절차는 다음과 같습니다.

1. **발급 (매장 등록 시 자동)** — 본사(`hq_admin`)가 관리자 화면(`/admin.html`)에서 `POST /api/admin/stores`로
   매장을 등록하면 `posToken`이 자동으로 함께 발급됩니다. 대량 등록(`/bulk`)도 동일합니다.
2. **직원 입력 (최초 1회)** — 매장 직원이 POS 탭앱을 처음 열면 토큰 입력 화면이 뜹니다. 본사 또는
   매장관리자가 관리자 화면의 **POS 토큰 관리** 표에서 해당 매장 토큰을 "보기"/"복사"해 전달하면, 직원이
   그 값을 한 번 입력합니다. 입력한 토큰은 POS 단말기의 `localStorage`에 저장되어 이후에는 다시
   묻지 않습니다.
3. **재발급(회전)** — 토큰이 유출됐거나(직원 퇴사, 단말기 분실 등) 정기 로테이션이 필요하면 `hq_admin`이
   관리자 화면에서 **재발급** 버튼(`POST /api/admin/stores/:id/pos-token`)을 누릅니다. **재발급 즉시 기존
   토큰은 무효화되며, 그 토큰을 쓰던 모든 POS 단말기는 다음 요청부터 401(`INVALID_STORE_TOKEN`)을 받고
   자동으로 토큰 재입력 화면으로 돌아갑니다** — 그러므로 재발급 직후 매장에 새 토큰을 다시 전달해야
   대기열 화면이 끊기지 않습니다. `store_admin`은 조회/복사만 가능하고 재발급 권한은 없습니다(`hq_admin` 전용).
4. **로컬 개발** — 자동 시드되는 테스트 매장(`merchantId '0'`)은 최초 생성 시 토큰이 비어 있을 수 있으므로,
   관리자 화면에서 한 번 재발급을 눌러 값을 채워야 합니다. 자세한 절차는
   [`pos-plugin/README.md`의 "로컬 미리보기"](../pos-plugin/README.md#로컬-미리보기) 참고.

<details>
<summary>멀티 가맹점(매장) 구조 — merchantId ↔ store_id, 2계층 관리자</summary>

- 예약/결제 API는 `merchantId`(토스 SDK가 단말기에서 넘겨주는 `sdk.merchant.id`)를 필수로 받습니다.
  등록되지 않은 `merchantId`면 404를 반환합니다 — 본사가 `/api/admin/stores`로 먼저 매장을 등록해야
  그 매장의 플러그인 요청이 통과됩니다.
- 모든 예약/결제 레코드는 내부 `storeId`로 스코프됩니다(Postgres/SQLite, Prisma). 대기번호(`queueNumber`)도
  매장별·날짜별로 원자적으로 독립 채번되고, 결제 화면의 전화번호→예약 매칭(`findLatestReservationByPhone`)도
  같은 매장 안에서만 찾습니다.
- 로컬 개발/미리보기는 `front-plugin/sdk.js`의 `merchant.id: 0` 오버라이드와 짝을 맞춰 서버가 부팅 시
  `merchantId: '0'` 테스트 매장을 자동으로 시드해둡니다.
- **관리자 인증은 이메일/비밀번호 + JWT 2계층 구조**입니다. `hq_admin`(본사)은 전체 매장을 보고 매장/매장관리자를
  등록할 수 있고, `store_admin`(가맹점)은 로그인하는 순간 자기 `storeId`로 강제 스코프되어 다른 매장의
  예약/결제는 조회·조작(`/complete`, `/call`, `DELETE`)이 전부 403으로 막힙니다. 최초 부팅 시 `hq_admin` 계정이
  없으면 자동 생성됩니다(`backend/.env.example`의 `ADMIN_BOOTSTRAP_*` 참고).
- `admin.html` 상단에 매장 선택 드롭다운("전체 매장" 포함, `hq_admin`만 보임)과 "매장 등록"/"매장 관리자 계정 발급"
  폼이 있습니다. `store_admin`으로 로그인하면 이 드롭다운/폼은 숨겨지고 자기 매장 화면만 보입니다.
- 자세한 아키텍처와 남은 로드맵(Phase 4 대량 온보딩, Phase 5 스케일 검증)은
  [`multi-store-architecture-review.md`](./multi-store-architecture-review.md)와
  [`01-plan/features/multi-store-support.plan.md`](./01-plan/features/multi-store-support.plan.md) 참고.

</details>

<details>
<summary>세부 동작 참고</summary>

- `serviceType`은 `엔진오일 교체`/`정기점검`/`타이어 교체·펑크수리`/`배터리 교체`/`브레이크 정비`/`기타 수리·상담` 중 하나를 가리키는 키
  (`oil`/`inspection`/`tire`/`battery`/`brake`/`etc`)입니다. 서버([server.js](../backend/server.js)의 `SERVICE_TYPES`), 토스프론트 예약 화면
  ([front-plugin/reservation.html](../front-plugin/reservation.html)), 관리자 페이지([admin.html](../backend/public/admin.html)) 세 군데에
  같은 목록이 하드코딩돼 있어서, 항목을 추가/변경할 땐 세 파일을 함께 고쳐야 합니다.
- 관리자 페이지(`/admin.html`)에는 실제 손님 없이 예약을 넣어볼 수 있는 "테스트 예약 생성" 폼이 있습니다 — 대기인원 안내와
  수동 호출 동작을 확인할 때 씁니다.
- `POST /api/payments`에 `paymentKey`(토스프론트에서 `sdk.payment.requestPayment()` 호출 시 발급한 값)를 같이 보내면,
  같은 `paymentKey`로 재요청해도 기존 레코드를 그대로 반환합니다 (영수증 중복 발송 방지, 결제 화면 네트워크 재시도 대비).
- 알림톡을 보냈다고 자동으로 대기열 "앞사람"에서 빠지는 게 아닙니다 — 정비가 실제로 끝나면 관리자 페이지에서 그 예약의
  "완료" 버튼을 눌러야(`POST /api/reservations/:id/complete`) 다음 예약이 "앞에 아무도 없음"으로 계산됩니다.
- 결제 화면에서 차량번호를 다시 입력받지 않는 대신, 전화번호로 `findLatestReservationByPhone`가 그 손님의 예약 기록을 찾습니다
  (같은 날 예약을 우선하고, 없으면 그 번호로 등록된 가장 최근 예약을 사용). 예약 없이 바로 결제하러 온 손님은 매칭되는 기록이 없어
  차량번호/정비항목 없이 영수증이 나갑니다.
- 예약 상태값은 `waiting`(대기중)/`called`(호출완료)/`notify_failed`(알림실패)/`completed`(정비완료)/`cancelled`(취소됨) 5가지,
  결제 상태값은 `requested`/`receipt_sent`/`receipt_failed`/`cancelled` 4가지입니다. 관리자 화면에서 예약/결제를 취소하면
  알림톡 발송 실패(`notify_failed`/`receipt_failed`) 건도 "재시도" 버튼으로 즉시 재발송을 시도할 수 있습니다(위 API 표 참고).
- 관리자 로그인은 IP 기준 레이트리밋 외에 **같은 계정이 비밀번호를 5회 연속 틀리면 15분 잠금**됩니다(계정 존재 여부는
  여전히 노출하지 않도록, 존재하지 않는 이메일은 이전과 동일하게 401만 돌려줍니다).
- 예약의 알림톡 실패 신호는 두 종류입니다 — `status: 'notify_failed'`(순서 호출 알림톡 실패, 예약 상태 자체가 바뀜)와
  `intakeNotifyStatus: 'failed'`(접수 시 대기번호 안내 알림톡 실패, 예약 상태와 무관하게 별도 컬럼으로만 추적). 접수
  알림이 성공하면 이 컬럼엔 아무것도 쓰지 않고 `null`을 유지합니다 — 해피패스에 DB 쓰기를 늘리지 않기 위해서입니다.
  두 실패 모두 `GET /api/reservations/failed`에서 함께 조회되고, 관리자 화면은 `status`/`intakeNotifyStatus` 값을 보고
  `retry-notify`/`retry-intake` 중 맞는 재발송 API를 고릅니다.
- POS 대기열은 오늘 접수분과 함께 아직 안 끝난 **이월 건**(예: 밤새 맡겨둔 차의 어제 `called` 예약)도 함께 보여줍니다.
  단, 어제 `waiting`(즉 어제 안에 한 번도 호출되지 않은 손님)은 노쇼로 보고 목록에서 빼고, 호출(`call`)은 여전히 오늘
  접수분만 가능합니다 — 그래야 "어제 대기 손님을 오늘 실수로 새로 호출"하는 사고를 계속 막을 수 있습니다.
- `GET /api/admin/summary` 응답은 `{ ok, date, storeId, reservations: {total, waiting, called, notify_failed, completed,
  cancelled}, payments: {total, amountSum, receiptFailed}, intakeFailed }` 형태입니다. 예약은 `serviceDate` 기준, 결제는
  `createdAt`의 KST 하루 범위 기준으로 집계하며, 관리자 화면 요약 카드의 "발송실패 F건"은
  `reservations.notify_failed + intakeFailed`(순서 호출 실패 + 접수 알림 실패)를 더한 값입니다.

</details>

## 알림톡 템플릿 변수

솔라피 알림톡 템플릿에서 쓰는 치환 변수다. 카카오 채널(`SOLAPI_KAKAO_PFID`)과 템플릿 ID가 비어 있으면 일반 문자(SMS)로 자동 폴백한다.

| 변수 | 설명 |
| --- | --- |
| `#{차량번호}` | 예) 12가3456 |
| `#{전화번호}` | 하이픈 없는 숫자만 |
| `#{대기번호}` | 예약 순번 (예약/순서호출 템플릿) |
| `#{대기인원}` | 내 앞에 대기중인 인원수 (예약 접수 템플릿 전용) |
| `#{정비항목}` | 예약 시 선택한 정비 종류 (예: 엔진오일 교체). 예약 접수/순서호출/영수증 템플릿에서 사용. 영수증에서는 전화번호로 찾은 예약 기록에서 가져오며, 매칭되는 예약이 없으면 빈 값 |
| `#{결제금액}` | 예) "15,000원" (영수증 템플릿) |
| `#{수신거부}` | 무료수신거부 안내 문구(`PROMO_OPT_OUT_TEXT`, 기본값은 [개인정보·광고성 정보 처리](./privacy.md) 참고). 프로모션(광고) 템플릿 전용 |
