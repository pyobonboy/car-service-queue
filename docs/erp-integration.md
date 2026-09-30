# 정비 전산(ERP) 연동 — Open API 주문 생성 (1차 구현)

> 현재 주 경로는 POS 플러그인 장바구니 방식(`/api/erp/carts`)이다. 경위는 [ERP연동-결제경로-조사결과](./ERP연동-결제경로-조사결과.md), 설계는 [전산-POS-장바구니-연동설계](./전산-POS-장바구니-연동설계.md)에 있다. 아래 Open API 주문 생성(`/api/erp/draft-orders`)은 1차 구현 기록이다.

정비 전산에서 "물건 담기"를 누르면 해당 매장의 토스 POS에 **미결제(OPENED) 주문이 자동 생성**되어,
매장 직원은 **결제만** 진행하면 되게 하는 연동입니다. 상세 스펙은 비공개 연동 스펙 문서
(v0.3, 1부는 전산 개발 담당자용 기술 명세, 2부는 비개발자용 쉬운 설명)를 참고하세요.
**상태: 1차 구현 완료.** 실호출로 주문 생성까지는 확인했지만(2026-08-26), 이 방식으로 만든 주문은 POS에서
결제할 수 없다는 사실이 실단말기에서 확인되어 주 경로가 장바구니 방식으로 바뀌었습니다. 아직 실제 매장에서
시연해보지 않았습니다.

```
[정비 전산] --(1) 주문 전송(X-ERP-Token)--> [이 서버] --(2) 주문 생성--> [토스 POS, OPENED]
                                                                      └-> 매장 직원이 결제만 진행
```

1. 전산이 `POST /api/erp/draft-orders`로 상품 목록(이름·가격·수량)과 매장 코드(`storeCode`)를 보냅니다.
2. 이 서버가 `storeCode`로 매장을 찾아(§매장 코드 등록은 아래 참고) 토스 Open API 주문 생성을 호출합니다.
   상품은 토스 POS 카탈로그에 사전 등록할 필요가 없습니다 — `targetType: "AD_HOC"`로 전산이 보낸
   이름·가격 그대로 임의 상품 라인아이템이 만들어집니다.
3. 생성된 주문은 결제 정보 없이 **OPENED(미결제)** 상태로 토스 POS **[현황] 탭**에 표시되고,
   매장 직원은 그 주문을 선택해 결제만 하면 됩니다.
4. 결제가 실제로 완료됐는지를 전산에 자동으로 알려주는 콜백(2단계)은 **아직 구현되지 않았습니다**
   (전산 측이 `GET /api/erp/draft-orders/:referenceId`로 재조회하는 방식만 지금 가능).

## 토스 공식 확인 사항 (2026-08 승인 회신)

토스 주문 생성 API 사용 승인을 받으면서 개발자센터로부터 직접 확인받은 내용입니다. **공개 문서만
보면 오해하기 쉬운 지점**이라 여기 명시해둡니다 — 아래와 다르게 구현하면 안 됩니다.

- **`payments`는 필수 필드입니다.** 미결제 주문이라도 `"payments": []`(빈 배열)을 반드시 보내야 합니다.
  공개 문서는 마치 생략하면 OPENED로 생성되는 것처럼 읽히지만 **틀렸습니다** — 생략하면 400이 납니다.
- `requestedInfo`를 전달하지 **않으면** 주문이 OPENED(미결제) 상태로 생성됩니다.
- 카탈로그에 없는 상품은 `targetType: "AD_HOC"` + 상품 정보(`item`)를 함께 전달합니다.
- 아래 네 가지는 **실제 토스 서버로 호출해가며 400(errorCode 4000)을 하나씩 잡아 확정**한 값입니다
  (2026-08-26). 공개 문서만 보고 추측하면 틀리기 쉬운 지점이라 그대로 지켜야 합니다.

| 필드 | 확정값 | 비고 |
| --- | --- | --- |
| `lineItems[].diningOption` | **`HERE`** | `FOR_HERE`는 4000 에러. 환경변수 `TOSS_DINING_OPTION`으로 조정 가능(기본 `HERE`) |
| `lineItems[].item.category` | **필수**, `{ title }` 객체 | enum이 아니라 자유 문자열이라 카탈로그 사전 등록 불필요. 기본 `'정비'` |
| `lineItems[].itemPrice.title` | **필수** | 가격 항목명. 단일 가격이면 `'기본'` |
| `order.chargePrice` | **숫자가 아니라 객체** | `{ listPrice, discountAmount, tipAmount, serviceChargeAmount, taxAmount, supplyAmount, taxExemptAmount, totalAmount }`. 부가세 포함가 기준 `taxAmount = round(총액/11)`, `supplyAmount = 총액 - taxAmount` |

- 위 body 구성의 참고 구현이자 실호출 검증 도구는 [`backend/scripts/verify-toss-order.js`](../backend/scripts/verify-toss-order.js)입니다.

## 매장 코드 등록

전산이 보내는 `storeCode`는 우리 쪽 매장(`Store`)과 미리 매핑되어 있어야 합니다. `hq_admin`이 관리자
화면에서 매장마다 `POST /api/admin/stores/:id/erp-code`를 한 번 호출해 `erpStoreCode`를 등록해두면,
이후 전산이 그 코드로 보낸 요청이 해당 매장으로 연결됩니다. 등록되지 않은 코드로 요청이 오면 404가
반환됩니다 — 새 매장을 연동할 때마다 이 등록이 먼저 필요합니다.

필요한 환경변수(`TOSS_OPENAPI_ACCESS_KEY`/`TOSS_OPENAPI_SECRET_KEY`/`ERP_API_TOKEN`/`TOSS_DINING_OPTION`)는
[`backend/.env.example`](../backend/.env.example)에 설명과 함께 정리되어 있습니다.

**전산 트래픽이 몰릴 때 만질 수 있는 손잡이**도 [`backend/.env.example`](../backend/.env.example)에 있습니다 —
`ERP_STORE_LIMIT_PER_MIN`(매장 하나당 분당 요청 상한, 기본 120)과 `ERP_IP_LIMIT_PER_MIN`
(그걸 우회하는 폭주를 막는 IP 기준 백스톱, 기본 3000)입니다. 정상적인 매장 요청까지 429로
막히는 것 같으면 이 두 값을 먼저 확인·조정하세요.

