# API QA Portfolio — DummyJSON Test Automation

Postman + Newman 기반 API 테스트 자동화 프로젝트입니다.
회원 인증(로그인) → 토큰 발급 → 인증된 요청 → 상품 CRUD로 이어지는 흐름을 설계하고,
Positive / Negative / Edge Case 테스트를 자동화했습니다.

---

## 1. 프로젝트 목적 및 대상 API

- **목적**: Postman을 활용한 API 테스트 설계, 환경 변수/체이닝 구성, 자동 검증 스크립트 작성,
  Newman CLI를 통한 CI 친화적 리포트 생성 역량을 보여주기 위한 포트폴리오입니다.
- **대상 API**: [DummyJSON](https://dummyjson.com) — 인증(JWT), 상품 CRUD 등 REST 엔드포인트를 제공하는
  공개 목업(Mock) API입니다.
- **테스트 계정**: `username: emilys / password: emilyspass` (DummyJSON 공식 문서 제공 테스트 계정)

---

## 2. 테스트 전략

### 2.1 범위 (Test Scope)

| 구분 | 내용 |
|---|---|
| 기능 테스트 (Happy Path) | 로그인, 인증 확인, 상품 조회/생성/수정/삭제 |
| 인증/인가 | 로그인 성공 시 토큰 발급 및 후속 요청 체이닝 |
| 예외 테스트 (Negative) | 잘못된 비밀번호, 존재하지 않는 리소스(404), 인증 우회 케이스 |
| 경계값 테스트 (Edge Case) | limit=0 파라미터, 대용량/다국어 페이로드 |

### 2.2 Collection 구조

```
DummyJSON API Test Suite
├── 01_Auth
│   └── POST Login (Valid)
├── 02_Product_CRUD
│   ├── GET Auth Me (Token Check)
│   ├── GET Product by ID
│   ├── POST Add Product
│   ├── PUT Update Product
│   └── DELETE Product
├── 03_Negative_Tests
│   ├── POST Login (Invalid Password)
│   ├── GET Non-existent Product (404)
│   └── GET Auth Me Without Token (Cookie Bypass Found)
└── 04_Edge_Cases
    ├── GET Products - Boundary Limit
    └── POST Add Product - Large Payload
```

총 **11개 요청 / 18개 assertion**으로 구성되어 있습니다.

### 2.3 Positive / Negative 비율

- Positive (정상 케이스): 6개
- Negative (예외 케이스): 3개
- Edge Case (경계값): 2개

정상 케이스만이 아니라 예외 상황과 경계값까지 균형 있게 검증하도록 설계했습니다.

---

## 3. 환경 변수 및 체이닝(Chaining) 구성

Environment: `DummyJSON - Dev`

| Variable | 설명 |
|---|---|
| `base_url` | `https://dummyjson.com` — 서버 주소 하드코딩 제거 |
| `access_token` | 로그인 성공 시 응답에서 자동 추출/저장 |
| `user_id` | 로그인한 사용자 ID 저장 |
| `created_product_id` | 상품 생성 시 응답 ID 저장 (후속 시나리오 확장용) |

**체이닝 흐름**: `POST Login` 요청의 Post-response 스크립트에서 `accessToken`, `id`를 환경 변수로 저장 →
이후 요청들은 Authorization 탭에서 `{{access_token}}`을 Bearer Token으로 사용하여 인증된 요청을 수행합니다.

```javascript
// POST Login (Valid) - Post-response Script
const response = pm.response.json();
pm.environment.set("access_token", response.accessToken);
pm.environment.set("user_id", response.id);
```

---

## 4. 자동 검증 스크립트 (Tests)

모든 요청에 상태 코드 검증과 응답 필드 검증을 최소 1~2개씩 포함했습니다. 예시:

```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Response has accessToken", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property("accessToken");
});
```

---

## 5. Newman 실행 방법

### 설치
```bash
npm install -g newman
npm install -g newman-reporter-htmlextra
```

### 실행
```bash
newman run "collections/DummyJSON API Test Suite.postman_collection.json" \
  -e "environments/DummyJSON - Dev.postman_environment.json" \
  -r cli,htmlextra \
  --reporter-htmlextra-export "./reports/report.html" \
  --reporter-htmlextra-title "QA Portfolio API Test Report" \
  --reporter-htmlextra-darkTheme
```

실행 결과는 `/reports/report.html`에서 확인할 수 있습니다.

---

## 6. 테스트 결과 요약

| 항목 | 결과 |
|---|---|
| 총 요청 | 11 |
| 총 assertion | 18 |
| Pass | 17 |
| Fail | 1 (의도적으로 문서화한 발견 사항, 아래 참고) |

---

## 7. 발견한 이슈 (Findings)

테스트 과정에서 예상과 다른 동작을 발견하여 아래와 같이 문서화했습니다.

### NEG-003: 인증 토큰 없이 `/auth/me` 호출 시 인증 우회 발견

| 항목 | 내용 |
|---|---|
| 시나리오 | Authorization 헤더 없이 `GET /auth/me` 호출 |
| 예상 결과 | `401 Unauthorized` |
| 실제 결과 | `200 OK` (이전 로그인 요청에서 발급된 세션 쿠키로 인증이 우회됨) |
| 원인 분석 | Postman이 이전 로그인 응답의 쿠키를 자동으로 저장/전송하여, Bearer 토큰이 없어도 쿠키 기반 세션으로 인증이 유지됨 |
| 권장 조치 | 테스트 환경에서는 요청 간 쿠키 격리(Cookie Jar 초기화) 필요. 서비스 측면에서는 토큰 전용 인증 정책인지, 쿠키 기반 인증도 허용하는지 명확히 정의 필요 |

이 케이스는 버그가 아니라 **테스트 환경 설정과 인증 정책 사이의 경계에서 발생한 발견 사항**으로 판단하여,
실패로 남겨두고 원인과 함께 기록했습니다.

---

## 8. 폴더 구조

```
api-qa-portfolio/
├── README.md
├── collections/
│   └── DummyJSON API Test Suite.postman_collection.json
├── environments/
│   └── DummyJSON - Dev.postman_environment.json
├── reports/
│   ├── report.html
│   └── screenshots/
└── docs/
    └── test-cases.md
```

---

## 9. 배운 점 / 트러블슈팅

- **Environment 저장 누락 이슈**: Postman에서 변수 값을 입력한 후 저장(Ctrl+S) 없이 Export하면,
  마지막 저장 시점의 (비어있는) 값이 그대로 내보내져 `Invalid URI` 에러가 발생함을 확인. 이후
  "수정 → 저장 확인(탭의 미저장 표시 확인) → Export" 순서를 습관화하여 해결.
- **Newman `--env-var` 옵션 활용**: Export 파일의 상태를 신뢰할 수 없는 상황에서, 커맨드라인에서
  직접 `--env-var "key=value"`로 변수를 주입하여 문제를 우회하고 원인을 좁혀나가는 방식으로 디버깅.
- **Positive 테스트만으로는 부족함**: 인증 우회처럼 예상과 다른 동작은 Negative 테스트를 설계해야만
  발견할 수 있었음. 정상 케이스 검증과 별개로 예외/경계 케이스 설계의 중요성을 체감.
