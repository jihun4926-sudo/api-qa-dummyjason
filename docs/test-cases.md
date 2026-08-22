# 테스트 케이스 문서 (Test Cases)

DummyJSON API Test Suite에 포함된 전체 테스트 케이스입니다.
각 케이스는 시나리오, 예상 결과, 실제 결과, Pass/Fail 여부를 기록했습니다.

---

## 01_Auth

| Case ID | 요청 | 시나리오 | Method / Endpoint | 예상 결과 | 실제 결과 | 결과 |
|---|---|---|---|---|---|---|
| AUTH-001 | POST Login (Valid) | 유효한 아이디/비밀번호로 로그인 | `POST /auth/login` | 200 OK, accessToken 반환 | 200 OK, accessToken 정상 반환 | ✅ Pass |

**검증 항목**
- Status code is 200
- Response has accessToken

**체이닝**: 응답의 `accessToken`, `id`를 환경 변수 `access_token`, `user_id`로 저장하여 이후 요청에서 재사용.

---

## 02_Product_CRUD

| Case ID | 요청 | 시나리오 | Method / Endpoint | 예상 결과 | 실제 결과 | 결과 |
|---|---|---|---|---|---|---|
| CRUD-001 | GET Auth Me (Token Check) | 발급받은 토큰으로 본인 정보 조회 | `GET /auth/me` | 200 OK, 로그인한 유저 정보와 id 일치 | 200 OK, id 일치 확인 | ✅ Pass |
| CRUD-002 | GET Product by ID | 특정 상품 상세 조회 | `GET /products/1` | 200 OK, id/title/price 필드 존재 | 200 OK, 필드 정상 확인 | ✅ Pass |
| CRUD-003 | POST Add Product | 신규 상품 생성 | `POST /products/add` | 201 Created, 입력한 title 그대로 반환 | 201 Created, title 일치 | ✅ Pass |
| CRUD-004 | PUT Update Product | 기존 상품 정보 수정 | `PUT /products/1` | 200 OK, title이 수정된 값으로 반영 | 200 OK, title 수정 확인 | ✅ Pass |
| CRUD-005 | DELETE Product | 상품 삭제 | `DELETE /products/1` | 200 OK, isDeleted: true | 200 OK, isDeleted true 확인 | ✅ Pass |

**검증 항목 (공통)**: Status code 검증 + 응답 본문의 핵심 필드 값 검증 (각 요청당 2개)

**참고**: DummyJSON은 실제 DB에 반영되지 않는 목업 API로, POST/PUT/DELETE 응답은 성공처럼 오지만 실제 데이터는 변경되지 않음.

---

## 03_Negative_Tests

| Case ID | 요청 | 시나리오 | Method / Endpoint | 예상 결과 | 실제 결과 | 결과 |
|---|---|---|---|---|---|---|
| NEG-001 | POST Login (Invalid Password) | 잘못된 비밀번호로 로그인 시도 | `POST /auth/login` | 400/401, 에러 메시지 포함 | 400 Bad Request, "Invalid credentials" | ✅ Pass |
| NEG-002 | GET Non-existent Product (404) | 존재하지 않는 상품 ID 조회 | `GET /products/9999` | 404 Not Found | 404 Not Found, 에러 메시지 포함 | ✅ Pass |
| NEG-003 | GET Auth Me Without Token (Cookie Bypass Found) | 인증 토큰 없이 사용자 정보 조회 | `GET /auth/me` | 401 Unauthorized | **200 OK** (세션 쿠키로 인증 우회됨) | ❌ Fail (의도적 발견 사항, 아래 상세 참고) |

### NEG-003 상세 — 발견된 이슈

Authorization 헤더를 제거(No Auth)하고 요청했음에도 401이 아닌 200 OK가 반환되었습니다.

- **원인**: 이전 로그인 요청에서 서버가 내려준 세션 쿠키를 Postman이 자동으로 저장하고 있다가,
  이후 같은 도메인 요청에 자동으로 실어 보내면서 Bearer 토큰 없이도 인증이 유지됨.
- **판단**: 애플리케이션 버그가 아니라 테스트 클라이언트(Postman)의 쿠키 자동 첨부 동작과
  서버의 쿠키 기반 인증 허용 정책이 맞물려 발생한 케이스로 판단.
- **권장 조치**: 테스트 시나리오 실행 전 쿠키 초기화(Cookie Jar Clear), 또는 서비스 측에서
  토큰 전용 인증인지 쿠키 병행 인증인지 정책을 명확히 문서화.
- **처리**: 실패로 남겨두고 원인과 함께 문서화 (README 7번 항목 참고).

---

## 04_Edge_Cases

| Case ID | 요청 | 시나리오 | Method / Endpoint | 예상 결과 | 실제 결과 | 결과 |
|---|---|---|---|---|---|---|
| EDGE-001 | GET Products - Boundary Limit | limit 파라미터를 0으로 요청 | `GET /products?limit=0` | 200 OK, 정상적으로 기본/빈 목록 처리 | 200 OK, products 필드 정상 반환 | ✅ Pass |
| EDGE-002 | POST Add Product - Large Payload | 매우 긴 텍스트 + 다국어(한글) 포함 페이로드 전송 | `POST /products/add` | 201 Created, 텍스트 깨짐 없이 정상 처리 | 201 Created, 한글/장문 텍스트 정상 반영 | ✅ Pass |

---

## 전체 요약

| 구분 | 개수 |
|---|---|
| 총 요청 수 | 11 |
| 총 assertion 수 | 18 |
| Pass | 17 |
| Fail (의도적 발견 사항 포함) | 1 |
| Positive 케이스 | 6 |
| Negative 케이스 | 3 |
| Edge Case | 2 |

---

## 실행 방법

```bash
newman run "collections/DummyJSON API Test Suite.postman_collection.json" \
  -e "environments/DummyJSON - Dev.postman_environment.json" \
  -r cli,htmlextra \
  --reporter-htmlextra-export "./reports/report.html" \
  --reporter-htmlextra-title "QA Portfolio API Test Report" \
  --reporter-htmlextra-darkTheme
```
